# tf2, Frames, and Simulation Time

Adapted from [`references/tf2-urdf.md`](https://github.com/dbwls99706/ros2-engineering-skills/blob/61870e4b94098d4d4f4b1f640936f9a3f7721304/references/tf2-urdf.md) and [`references/simulation.md`](https://github.com/dbwls99706/ros2-engineering-skills/blob/61870e4b94098d4d4f4b1f640936f9a3f7721304/references/simulation.md) in ros2-engineering-skills (Apache-2.0), with conventions from [REP 103](https://www.ros.org/reps/rep-0103.html) and [REP 105](https://www.ros.org/reps/rep-0105.html). APIs are checked against the Jazzy `tf2_ros` Python package.

## The frame tree

```text
map                      localization: AMCL, SLAM, or ground truth
 └── odom                odometry: continuous, drifts
      └── base_link      robot_state_publisher or the bridge
           ├── laser_link
           ├── imu_link
           └── camera_link
                └── camera_optical_frame   +Z forward, +X right, +Y down
```

- Each parent → child transform has exactly one publisher. Two publishers
  make lookups flicker between answers; tf2 does not report it.
- `odom` → `base_link` is continuous and never jumps; `map` → `odom` absorbs
  localization corrections.
- `base_footprint`, when present, is the ground projection of `base_link`.
- Frame IDs carry no leading slash. Multi-robot systems prefix them, such as
  `robot1/base_link`, and each robot's localization publishes
  `map` → `robot1/odom`.

REP 103 fixes meters, radians, and SI units; `x` forward, `y` left, `z` up
for bodies; and right-handed rotations. A stage authored in centimeters or
Z-forward assets must be converted before its poses reach ROS.

## Quaternions and cameras

`geometry_msgs/Quaternion` fields are named `x, y, z, w`, and a default
message is all zeros, which is not a rotation: set `w = 1.0` for identity.
Isaac core APIs return WXYZ arrays; OmniGraph and SciPy use XYZW.

A USD camera looks along `-Z` with `+Y` up. The ROS optical frame looks along
`+Z` with `+Y` down, a 180° rotation about `X`: quaternion
`(x, y, z, w) = (1, 0, 0, 0)` from the USD camera to the optical frame. The
body-style `camera_link` beside it follows REP 103, `x` forward.

## Broadcast

Static transforms go on `/tf_static` with transient-local durability, so late
subscribers receive them; publish them once. Moving transforms go on `/tf`
at the rate their source changes, stamped with the time of that state.

```python
def publish_mount(node, parent: str, child: str, xyz: tuple[float, float, float]) -> object:
    from geometry_msgs.msg import TransformStamped
    from tf2_ros import StaticTransformBroadcaster

    broadcaster = StaticTransformBroadcaster(node)
    transform = TransformStamped()
    transform.header.stamp = node.get_clock().now().to_msg()
    transform.header.frame_id = parent
    transform.child_frame_id = child
    transform.transform.translation.x, transform.transform.translation.y, transform.transform.translation.z = xyz
    transform.transform.rotation.w = 1.0
    broadcaster.sendTransform(transform)
    return broadcaster
```

The node owns the broadcaster's publisher, so dropping the Python object
does not remove it. Create one broadcaster per node and reuse it; to replace
it, `node.destroy_publisher(broadcaster.pub_tf)` first, or a second latched
publisher keeps serving the old transform.

## Look up

Create one buffer and listener when the process starts and reuse them; a
new buffer starts empty, so a lookup through a fresh one always fails.

```python
def start_tf_listener() -> tuple[object, object]:
    from tf2_ros import Buffer, TransformListener

    buffer = Buffer()
    return buffer, TransformListener(buffer, None, spin_thread=True)


def base_in_map(buffer, node) -> object | None:
    import rclpy.time
    from tf2_ros import TransformException

    try:
        return buffer.lookup_transform("map", "base_link", rclpy.time.Time())
    except TransformException as error:
        node.get_logger().warn(f"map -> base_link unavailable: {error}")
        return None
```

`TransformListener(buffer, None, spin_thread=True)` creates its own node and
thread, so the buffer fills while the simulator thread steps. Its internal
node does not use sim time; the stamps it stores are the publishers' stamps.
The thread is not a daemon: at the end of the run call
`listener.executor.shutdown()`, join `listener.dedicated_listener_thread`,
then `listener.node.destroy_node()`, or the process does not exit cleanly.

- `rclpy.time.Time()` asks for the latest common time of the chain; pass the
  message stamp when transforming sensor data taken at that instant.
- A `timeout` blocks the calling thread. From the thread that steps the
  simulator, look up without a timeout and retry after the next step.
- `can_transform` answers without raising; the exception text says whether
  a frame is unknown, disconnected, or outside the buffer's time window.
- The engine has no `tf2_geometry_msgs`, so `buffer.transform()` cannot
  convert messages there. Apply the returned rotation and translation with
  NumPy or SciPy, or transform in a ROS service image.

## Simulation time

With `use_sim_time: true`, a node's clock follows `/clock`. Set it on every
node that consumes simulated data, in launch files and parameter YAML alike,
and publish `/clock` from the simulator's time. In Python:

```python
def sim_time_node(name: str) -> object:
    import rclpy
    from rclpy.parameter import Parameter

    if not rclpy.ok():
        rclpy.init()
    return rclpy.create_node(name, parameter_overrides=[Parameter("use_sim_time", value=True)])
```

- Before the first `/clock` message, a sim-time clock reads zero. Wait for
  it before stamping or looking up.
- Timers and rates on a sim-time node advance only while the simulation
  plays; a paused simulation pauses them.
- Mixing clocks produces extrapolation errors: a sim-time consumer asking for
  a wall-clock-stamped transform, or the reverse, looks decades apart.
- Bag playback for sim-time consumers needs `ros2 bag play --clock`.

## URDF and robot_state_publisher

`robot_state_publisher` reads `robot_description` and `/joint_states`. It
publishes fixed joints on `/tf_static` and moving joints on `/tf`; a moving
joint missing from `/joint_states` leaves its child links out of the tree.
Expand xacro in the ROS image (`xacro robot.urdf.xacro > robot.urdf`) or in
the launch file.

With Isaac, choose one source for link transforms: `robot_state_publisher`
fed by the bridge's joint states, or the bridge's transform tree. Joint names
in `/joint_states` must match the URDF exactly; an imported USD robot keeps
the names its URDF import gave it.

## Debug

In a ROS service image:

```bash
ros2 run tf2_ros tf2_echo map base_link
ros2 run tf2_tools view_frames
ros2 topic echo /tf_static --once
```

The engine lacks `tf2_tools` and the `tf2_ros` executables; there, echo
`/tf` and `/tf_static` or call `can_transform` from Python.

| Symptom | Cause |
|---|---|
| "Lookup would require extrapolation into the future" | Asked for a stamp newer than the latest transform: clocks differ, or the publisher lags |
| "...into the past" | The stamp is older than the buffer's cache, 10 s by default |
| "Could not find a connection" | Two disconnected trees, usually a missing `map` → `odom` or `odom` → `base_link` |
| Frame flickers between poses | Two publishers for the same transform |
| Camera data rotated 90° or mirrored | Optical frame missing or USD camera axes used directly |
| Links missing below a joint | That joint absent from `/joint_states` |
| Stamps near zero or 1970 | Sim-time clock read before `/clock` arrived |

Use the [bridge guide](isaac-ros2-bridge.md) to publish the simulator's clock and transforms, and [Nav2](nav2.md) or [MoveIt 2 and ros2_control](moveit2-ros2-control.md) for the consumers that depend on them. Return to [ROS 2](../SKILL.md#references) for where ROS runs and the other guides.
