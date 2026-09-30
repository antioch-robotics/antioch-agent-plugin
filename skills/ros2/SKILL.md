---
name: ros2
version: "1.1.0"
description: Use for ROS 2 nodes, launch files, parameters, and message flow that drive or observe an Antioch simulation, including rclpy in the Isaac Sim engine, the Isaac ROS 2 bridge, simulation time, tf2, QoS and discovery, Nav2, MoveIt 2, and ros2_control, on Jazzy or another distro in a ROS service image. Service routes belong to antioch-platform and scene code to isaac-sim-6.
---

# ROS 2 on Antioch

Isaac Sim 6.0.1 bundles **ROS 2 Jazzy** (Ubuntu 24.04, Python 3.12), so
everything inside the simulator is Jazzy: the bridge, `rclpy`, and the
message packages. ROS service images next to it can run another
distribution; [distributions](#distributions) covers what that costs. Load
[Isaac Sim](../isaac-sim-6/SKILL.md) for scene code and
[Antioch platform](../antioch-platform/SKILL.md) for services and sessions;
its [ROS 2 guide](../antioch-platform/references/ros2.md) owns discovery and
routes between services.

The references adapt community and NVIDIA guidance to this runtime; each
names its upstream source. Concepts apply to every ROS 2 distribution;
facts marked Jazzy are checked against Jazzy only.

## Where ROS runs

| Place | What it has | Use it for |
|---|---|---|
| Engine service | In-process Jazzy Python bundled with Isaac, the Isaac bridge extensions, `ROS_DISTRO=jazzy`, `RMW_IMPLEMENTATION=rmw_fastrtps_cpp` | Bridge graphs, sim-side nodes, scenario checks that read or publish topics |
| ROS service image | `FROM ros:jazzy` (or another distro) pinned by digest, apt `ros-<distro>-*` packages, built workspaces, C++ | Nav2, MoveIt 2, ros2_control, `robot_state_publisher`, bags, custom interfaces |

The Jazzy engine carries `rclpy`, `tf2_ros` (Python), `launch`/`launch_ros`,
`message_filters`, `nav2_simple_commander`, `sensor_msgs_py`, and common
interfaces: `std`, `geometry`, `sensor`, `nav`, `trajectory`, `tf2`,
`visualization`, `vision`, `diagnostic`, `ackermann`, `nav2_msgs`, and
`simulation_interfaces`. Its `ros2` command has the `topic`, `node`, `pkg`,
`run`, and `doctor` verbs but not `service`, `param`, `action`, `interface`,
`launch`, `bag`, or `control`. It has no `moveit_msgs`, `control_msgs`,
`tf2_geometry_msgs`, rosbag2, `tf2_tools`, colcon, or compiler: use those
from a ROS service image. A custom interface the engine must load needs type
support built for its bundled Jazzy and Python 3.12; otherwise keep the node
that uses it in the ROS image.

Exec and health checks skip image entrypoints, so source ROS in the command:

```bash
antioch service exec --service planner -- bash -lc 'source /opt/ros/$ROS_DISTRO/setup.bash && ros2 topic list'
```

## Distributions

Mixing distributions in one graph is officially unsupported. A Humble
service talking to the Jazzy engine often works for common message types
over Fast DDS, but test the exact topics, services, and actions in both
directions before relying on it. Prefer Jazzy for new service images. Check
an API against the documentation of the distribution the service image
runs, and say which one you used.

| | Humble | Jazzy | Kilted |
|---|---|---|---|
| Support | LTS to May 2027 | LTS to May 2029 | Non-LTS |
| Nav2 `cmd_vel` | `Twist` | `Twist`; `TwistStamped` with `enable_stamped_cmd_vel` | `TwistStamped` by default |
| Nav2 behavior trees | BehaviorTree.CPP 3 | BehaviorTree.CPP 4, `BTCPP_format="4"` | BehaviorTree.CPP 4 |
| MoveIt 2 documentation | Its own `humble` tree | Rolling tree only | Rolling tree only |

## Python inside the engine

Import `rclpy` and message modules inside functions, like simulator imports,
so project modules import where no ROS is installed. The bridge keeps its
own context; initialize `rclpy` once per process and check `rclpy.ok()`
first, which matters in a Jupyter kernel that reruns a cell.

Never block the thread that steps the simulator: `rclpy.spin()` there stops
physics, `/clock`, and every bridge publisher. Spin once per step, or run an
executor on a daemon thread:

```python
def wait_for_scan(steps: int = 600) -> int:
    import antioch
    import rclpy
    from rclpy.qos import qos_profile_sensor_data
    from sensor_msgs.msg import LaserScan

    if not rclpy.ok():
        rclpy.init()
    node = rclpy.create_node("scan_check")
    scans: list[LaserScan] = []
    node.create_subscription(LaserScan, "scan", scans.append, qos_profile_sensor_data)
    world = antioch.world()
    try:
        for _ in range(steps):
            world.step(render=True)
            # spin_once runs at most one ready callback; drain a few per step
            for _ in range(4):
                rclpy.spin_once(node, timeout_sec=0.0)
            if scans:
                break
    finally:
        node.destroy_node()
    return len(scans)
```

A client that waits on the ROS graph, such as `BasicNavigator` or a TF
lookup with a timeout, deadlocks or times out when its wait needs data the
same thread would produce by stepping. Give it its own thread, or poll
without waiting between steps. A scenario that hangs with simulation time
frozen usually has a blocked stepping thread; a daemon watchdog thread that
logs simulation time and message counts every few seconds shows it before
the run's timeout does.

## Time, frames, and conventions

- Publish `/clock` from simulation time and set `use_sim_time: true` on every
  node that consumes simulated data. Timers and durations on a sim-time node
  advance only while the simulation plays.
- REP 103: meters, radians, SI units; `x` forward, `y` left, `z` up. REP 105:
  `map` → `odom` → `base_link`, and each transform has exactly one publisher.
- Message quaternions are fields named `x, y, z, w`. Isaac core APIs return
  WXYZ arrays, and OmniGraph `quatd` inputs are XYZW. Convert once at the
  boundary.
- A ROS camera optical frame looks along `+Z` with `+Y` down; a USD camera
  looks along `-Z` with `+Y` up.

[tf2 and time](references/tf2-and-time.md) covers lookups, static and
dynamic broadcasters, and the errors they raise.

## Evidence

A topic in `ros2 topic list` is not data flow. Check the publisher and
subscriber QoS with `ros2 topic info -v`, the rate with `ros2 topic hz`, and a
message's frame, stamp, and values. Check that `/clock` advances and that the
TF chain the consumer needs resolves at the message stamp. For navigation or
manipulation, measure the robot's simulated motion against the goal; an
accepted goal or a `SUCCEEDED` result is not proof that the robot arrived.
Record these as scenario checks through
[scenario design](../scenario-design/SKILL.md).

## References

| Task | Guide |
|---|---|
| Bridge graphs, sensors, clock, TF, odometry, simulation-control services | [Isaac ROS 2 bridge](references/isaac-ros2-bridge.md) |
| Frames, tf2 lookups, sim time, URDF and `robot_state_publisher` | [tf2 and time](references/tf2-and-time.md) |
| Nodes, executors, QoS, discovery failures, launch | [Nodes and QoS](references/nodes-and-qos.md) |
| Nav2 bringup, costmaps, localization, the simple commander | [Nav2](references/nav2.md) |
| MoveIt 2, ros2_control, topic-based hardware | [MoveIt 2 and ros2_control](references/moveit2-ros2-control.md) |

Use [research](../antioch-research/SKILL.md) for exact APIs and parameters.
`research_versions` shows which ROS corpora are live: `ros2-docs`,
`ros2-api`, `rclpy-source`, `nav2-docs`, `ros2-control-docs`, and
`moveit2-docs`. The docs corpora are Jazzy except MoveIt's, which tracks
Rolling. `rclpy-source` holds rclpy 7.1.12's docstrings, because
docs.ros.org renders Jazzy's rclpy API pages for nodes, parameters, and
modules empty. For another distribution, or when a corpus is missing, read
that distribution's versioned page at docs.ros.org, docs.nav2.org, or
control.ros.org, and say which you used.
