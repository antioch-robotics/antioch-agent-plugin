# Nav2 Against a Simulated Robot

Adapted from [`references/navigation.md`](https://github.com/dbwls99706/ros2-engineering-skills/blob/61870e4b94098d4d4f4b1f640936f9a3f7721304/references/navigation.md) in ros2-engineering-skills (Apache-2.0), NVIDIA's Jazzy [`carter_navigation`](https://github.com/isaac-sim/IsaacSim-ros_workspaces/tree/a9e8471ee901bc2332c1e4aca94ac580713ca3ab/jazzy_ws/src/navigation/carter_navigation) parameters (Apache-2.0), and the [Nav2 Jazzy documentation](https://docs.nav2.org/jazzy/). Simple-commander behavior is checked against Nav2's `jazzy` branch.

Nav2 runs in a ROS service image; the simulator supplies the robot. Map
generation from a scene belongs to Isaac Sim
[occupancy maps](../../isaac-sim-6/references/occupancy-map.md), and native
non-ROS navigation to Isaac Sim
[robot navigation](../../isaac-sim-6/references/isaac-sim-robot-navigation.md).

## The contract between simulator and Nav2

| Nav2 needs | Simulator side |
|---|---|
| `/clock` and `use_sim_time: true` on every Nav2 node | `ROS2PublishClock` from simulation time |
| `odom` → `base_link` on `/tf` | Odometry transform from the bridge, or `robot_localization` fusing sensors |
| Robot link frames, including the sensor frame | Bridge transform tree or `robot_state_publisher` |
| Odometry on the controller's `odom_topic` | `ROS2PublishOdometry` |
| `LaserScan` for AMCL | `ROS2RtxLidarHelper` with `laser_scan` |
| `LaserScan` or `PointCloud2` for costmaps | The same scan, a lidar point cloud, or depth point clouds |
| A consumer for `cmd_vel` | `ROS2SubscribeTwist` wired to the drive |
| A map for `map_server`, or SLAM | An occupancy map exported from the scene |
| `map` → `odom` | AMCL, SLAM Toolbox, or one static ground-truth transform |

On Jazzy, Nav2 publishes `geometry_msgs/Twist` on `cmd_vel` unless
`enable_stamped_cmd_vel` is true; `TwistStamped` is the default only from
Kilted on. Isaac's `ROS2SubscribeTwist` takes `Twist`.

## Bring it up

A ROS image built `FROM ros:jazzy` (pinned by digest) with
`ros-jazzy-navigation2` and `ros-jazzy-nav2-bringup` installed can run the
following; other distributions use their own `ros-<distro>-` packages:

```bash
ros2 launch nav2_bringup bringup_launch.py use_sim_time:=True map:=/workspace/project/nav/map.yaml params_file:=/workspace/project/nav/nav2_params.yaml
```

`bringup_launch.py` starts `map_server`, AMCL, and the navigation servers
under lifecycle managers; `slam:=True` swaps in SLAM Toolbox for the map
server and AMCL. `navigation_launch.py` alone starts the navigation servers
for a stack whose localization runs elsewhere. Sync or bake the map and
parameter files into the image; the path is inside the ROS service.

Start from the stock Jazzy `nav2_params.yaml` in `nav2_bringup` and change
what your robot differs in. It describes a TurtleBot: `base_footprint`
frames, a 0.22 m `robot_radius`, and the MPPI controller's limits for that
base. Jazzy plugin types use `::`, such as
`nav2_controller::SimpleGoalChecker`; slash-separated names come from older
releases and fail to load.

## Parameters that must match the simulated robot

- `use_sim_time: true` on every node, including `map_server`, AMCL, both
  costmaps, `velocity_smoother`, and `collision_monitor`.
- Frames: AMCL `base_frame_id`, `odom_frame_id`, `global_frame_id`; the
  navigator's and costmaps' `robot_base_frame`; the local costmap's
  `global_frame: odom` and the global costmap's `global_frame: map`.
- Topics: AMCL `scan_topic`; each costmap observation source's `topic` and
  `data_type` (`LaserScan` or `PointCloud2`); the controller's `odom_topic`.
- Geometry: `footprint` polygon or `robot_radius` in `base_link`, matching
  the collision geometry in the scene; `inflation_radius` larger than the
  inscribed radius.
- Motion limits: controller and `velocity_smoother` maximum velocities and
  accelerations at or below what the simulated drive achieves; a controller
  asking for more than the drive delivers oscillates or fails progress checks.
- Sensor ranges: `obstacle_max_range` and `raytrace_max_range` within the
  simulated lidar's range, or distant returns never clear the costmap.
- Sensor height: each observation source's `min_obstacle_height`, and the
  voxel layer's `origin_z`, against where scan points land in the costmap's
  frame. A simulated `odom` at a different height from the scan can put
  every point below the minimum, and the layer drops them silently.

A goal can succeed with an empty local costmap, so check the costmaps before
trusting a run: `ros2 topic echo --once /local_costmap/costmap` should show
occupied cells where the scan sees obstacles.

## Drive it from Python in the simulator (Jazzy)

`nav2_simple_commander` is in the Jazzy engine. `BasicNavigator` is a node that
spins itself inside every call, and with AMCL `waitUntilNav2Active()` keeps
republishing the navigator's initial pose until AMCL answers, which needs the
simulation to keep stepping. Run it on its own thread and step on the main
one, and bound, cancel, and clean up from the stepping side:

```python
def navigate_to(x: float, y: float, start: tuple[float, float] = (0.0, 0.0), localizer: str = "amcl", max_steps: int = 20_000, cancel_steps: int = 600) -> str:
    import threading

    import antioch
    import rclpy
    from geometry_msgs.msg import PoseStamped
    from nav2_simple_commander.robot_navigator import BasicNavigator
    from rclpy.parameter import Parameter

    def map_pose(px: float, py: float) -> PoseStamped:
        pose = PoseStamped()
        pose.header.frame_id = "map"
        pose.pose.position.x, pose.pose.position.y = px, py
        # The default quaternion is all zeros, which AMCL rejects
        pose.pose.orientation.w = 1.0
        return pose

    if not rclpy.ok():
        rclpy.init()
    navigator = BasicNavigator()
    navigator.set_parameters([Parameter("use_sim_time", value=True)])
    if localizer == "amcl":
        navigator.setInitialPose(map_pose(*start))
    stop = threading.Event()
    outcome: list[object] = []

    def drive() -> None:
        try:
            navigator.waitUntilNav2Active(localizer=localizer)
            if stop.is_set():
                return
            navigator.goToPose(map_pose(x, y))
            cancelled = False
            while not navigator.isTaskComplete():
                if stop.is_set() and not cancelled:
                    navigator.cancelTask()
                    cancelled = True
            outcome.append(navigator.getResult())
        except BaseException as error:  # handed to the stepping thread
            outcome.append(error)

    worker = threading.Thread(target=drive, daemon=True)
    worker.start()
    world = antioch.world()
    for _ in range(max_steps):
        world.step(render=True)
        if not worker.is_alive():
            break
    stop.set()
    # Cancelling needs Nav2 to answer, which needs the simulation to step
    for _ in range(cancel_steps):
        if not worker.is_alive():
            break
        world.step(render=True)
    if worker.is_alive():
        raise TimeoutError("Nav2 never became active; the waiting thread still owns the navigator")
    navigator.destroy_node()
    if outcome and isinstance(outcome[0], BaseException):
        raise outcome[0]
    return str(outcome[0]) if outcome else "timed out before a goal was sent"
```

`start` is the robot's pose in the map when the call begins. Pass
`localizer="robot_localization"` when no AMCL runs; that skips the
initial-pose wait. `waitUntilNav2Active()` has no timeout of its own, so a
stack that never activates leaves its thread waiting and the function
raises instead of destroying a node that thread still spins. A zero stamp on
a goal means "latest"; a wall-clock stamp against a sim-time stack fails TF
lookups.

`TaskResult.SUCCEEDED` means the goal checker accepted the pose inside its
tolerance. Measure the robot's simulated pose against the goal, the path
length, time, and contacts, and record them as scenario checks.

## Behavior trees

Jazzy's navigator runs BehaviorTree.CPP 4: custom XML uses v4 syntax and
`BTCPP_format="4"`. Humble's runs BehaviorTree.CPP 3, whose XML differs. Recovery behaviors in the default trees spin, back up,
and clear costmaps; in a scenario that measures a planner, record when they
fire, because a success after recovery is a different result.

## Failures

| Symptom | Check |
|---|---|
| Nodes listed but nothing happens | Lifecycle states; the lifecycle manager's log for the node that failed to configure |
| "Timed out waiting for transform" at startup | `/clock` advancing, `use_sim_time` everywhere, `map` → `odom` → `base_link` complete |
| AMCL never publishes a pose | Initial pose sent; `scan_topic` and scan QoS; map frame |
| Costmap empty | Observation `topic`, `data_type`, sensor frame in TF, ranges, `min_obstacle_height` and voxel `origin_z` against the scan's height |
| Costmap never clears | `clearing: true` and `raytrace_max_range`; scans with max-range returns |
| Planner fails at start | Robot inside an inflated obstacle; footprint larger than the real robot |
| Robot oscillates or stalls near goal | Controller limits above the drive's, tight goal tolerances, progress checker |
| `cmd_vel` published, robot still | Twist subscriber wiring, message type, drive controller in the scene |
| Robot spins in place unexpectedly | A recovery behavior fired; check `behavior_server` logs |

Use the [bridge guide](isaac-ros2-bridge.md) for the scan, odometry, and `cmd_vel` graph, [tf2 and time](tf2-and-time.md) for the frame chain, and [scenario checks](../../scenario-design/SKILL.md#measured-verdicts) to record arrival, path, and contacts. Return to [ROS 2](../SKILL.md#references) for where ROS runs and the other guides.
