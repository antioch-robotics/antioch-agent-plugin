# MoveIt 2 and ros2_control Against a Simulated Arm

Adapted from [`references/manipulation.md`](https://github.com/dbwls99706/ros2-engineering-skills/blob/61870e4b94098d4d4f4b1f640936f9a3f7721304/references/manipulation.md) and [`references/hardware-interface.md`](https://github.com/dbwls99706/ros2-engineering-skills/blob/61870e4b94098d4d4f4b1f640936f9a3f7721304/references/hardware-interface.md) in ros2-engineering-skills (Apache-2.0), NVIDIA's Jazzy [`isaac_moveit`](https://github.com/isaac-sim/IsaacSim-ros_workspaces/tree/a9e8471ee901bc2332c1e4aca94ac580713ca3ab/jazzy_ws/src/moveit/isaac_moveit) package and Isaac Sim 6.0.1 [`moveit.py`](https://github.com/isaac-sim/IsaacSim/blob/v6.0.1/source/standalone_examples/api/isaacsim.ros2.bridge/moveit.py) sample (Apache-2.0), and the [ros2_control Jazzy documentation](https://control.ros.org/jazzy/).

MoveIt, the controller manager, and the controllers run in a ROS service
image; the engine has no `moveit_msgs` or `control_msgs`. The simulator's
articulation is the hardware. Native Isaac motion generation without ROS
belongs to Isaac Sim [motion generation](../../isaac-sim-6/references/motion-generation.md).

MoveIt publishes documentation for Humble and Rolling, with no Jazzy tree.
Check a MoveIt API or parameter against the installed `ros-<distro>-moveit*` package
before relying on it.

## The loop

```text
move_group ── FollowJointTrajectory ──> joint_trajectory_controller
   ▲                                          │ position commands
   │ /joint_states                            ▼
joint_state_broadcaster <── states ── topic-based hardware (ros2_control)
                                              │ /isaac_joint_commands   ▲ /isaac_joint_states
                                              ▼                         │
                           Isaac: ROS2SubscribeJointState → IsaacArticulationController
                                  IsaacReadJointState → ROS2PublishJointState
```

The simulator side is the joint-state graph in the
[bridge reference](isaac-ros2-bridge.md#joint-state-and-joint-commands);
the topic names above are NVIDIA's convention, and any pair works when both
ends agree.

## Topic-based hardware

A ros2_control `System` plugin forwards commands to a topic and reads
states from another, so the controller manager treats the simulator like a
robot. On Jazzy, install ros-controls' plugin from apt,
`ros-jazzy-joint-state-topic-hardware-interface`, and confirm it with
`ros2 pkg list`. PickNik's `topic_based_ros2_control/TopicBasedSystem`, which
NVIDIA's `isaac_moveit` and MoveIt's Panda resources name, has no Jazzy apt
package; replace that plugin name in their URDFs.

```xml
<ros2_control name="IsaacArm" type="system">
  <hardware>
    <plugin>joint_state_topic_hardware_interface/JointStateTopicSystem</plugin>
    <param name="joint_commands_topic">/isaac_joint_commands</param>
    <param name="joint_states_topic">/isaac_joint_states</param>
    <param name="sum_wrapped_joint_states">true</param>
  </hardware>
  <joint name="joint1">
    <command_interface name="position"/>
    <state_interface name="position"><param name="initial_value">0.0</param></state_interface>
    <state_interface name="velocity"/>
    <state_interface name="effort"/>
  </joint>
</ros2_control>
```

Declare the `effort` state interface on every joint. Isaac's
`ROS2PublishJointState` fills `effort`, and the plugin deactivates the
hardware on its first read with "The requested state interface not found:
'joint1/effort'" when a joint lacks it.

- Joint names in the URDF, the controller YAML, the SRDF, and the Isaac
  articulation must match exactly.
- Isaac reports continuous-joint positions wrapped to ±2π. The
  ros-controls plugin's `sum_wrapped_joint_states` parameter unwraps them.
- Mimic joints, such as a parallel gripper's second finger, have state but no
  command interface. NVIDIA's Jazzy Panda keeps the hand out of ros2_control
  because the topic-based plugin's mimic handling published malformed joint
  commands; it bridges the gripper with a separate node. Test a mimic joint
  before trusting it.
- A position command drives the articulation's position targets, so drive
  stiffness and damping in USD decide how closely the arm tracks. Weak
  gains make the trajectory controller abort on path or goal tolerance.

## Controllers

```yaml
controller_manager:
  ros__parameters:
    update_rate: 100
    use_sim_time: true
    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster
    arm_controller:
      type: joint_trajectory_controller/JointTrajectoryController

arm_controller:
  ros__parameters:
    joints: [joint1, joint2, joint3, joint4, joint5, joint6]
    command_interfaces: [position]
    state_interfaces: [position, velocity]
```

Spawn controllers after the controller manager is up:
`ros2 run controller_manager spawner joint_state_broadcaster arm_controller`.
`ros2 control list_controllers` and `ros2 control list_hardware_interfaces`
show what loaded, what is active, and which interfaces are claimed. The
controller manager's clock handling with `use_sim_time` is on
control.ros.org under the controller manager's clocks page; read it before
tuning `update_rate` against a simulation that runs slower than real time.

For a mobile base, `diff_drive_controller` publishes odometry and, by
default, `odom` → `base_link`. Turn off either its TF (`enable_odom_tf:
false`) or the bridge's odometry transform; two publishers of one transform
break localization. Grippers use `parallel_gripper_action_controller` or the
older `gripper_action_controller`.

## MoveIt

`move_group` needs `robot_description`, `robot_description_semantic` (the
SRDF), kinematics, planning pipelines, joint limits, and a controller map:

```yaml
moveit_simple_controller_manager:
  controller_names: [arm_controller]
  arm_controller:
    type: FollowJointTrajectory
    action_ns: follow_joint_trajectory
    default: true
    joints: [joint1, joint2, joint3, joint4, joint5, joint6]
```

- Set `use_sim_time: true` on `move_group`, `robot_state_publisher`, the
  controller manager, and RViz.
- `moveit_configs_utils.MoveItConfigsBuilder` assembles these files from a
  MoveIt config package; the Setup Assistant generates the package. Pass
  `robot_description(file_path=...)` for the xacro that carries the
  `ros2_control` block; the default description can omit it, and the
  controller manager then waits forever for `robot_description`.
- Planning pipelines: OMPL for general planning, Pilz for deterministic
  lines and arcs, CHOMP and STOMP for optimization. Time parameterization
  runs as a response adapter; its velocity and acceleration scaling limits
  execution speed.
- A straight Cartesian line uses Pilz: load `pilz_industrial_motion_planner`
  as a pipeline, set `cartesian_limits` (`max_trans_vel`, `max_trans_acc`,
  `max_trans_dec`, `max_rot_vel`) under `robot_description_planning`, and
  plan with `planning_pipeline: pilz_industrial_motion_planner` and
  `planner_id: LIN` (`PTP` joint-space, `CIRC` arcs). LIN needs a start
  state at rest.
- The planning scene knows only what is published to it: add collision
  objects for tables and fixtures in the simulated scene, or feed a depth
  point cloud into the occupancy map monitor. A plan that ignores an object
  in the simulator is a missing scene object, not a planner fault.
- `MoveItPy` (`ros-jazzy-moveit-py`) runs in the ROS image, not the engine.
  With `use_sim_time: true` it has aborted on the `/clock` QoS-override
  parameter; drive `move_group` through its `move_action` action instead.
- Isaac Sim 6.0's `franka.usd` has no `panda_link8` body; its end effector is
  `panda_hand`. Its default `panda_joint4` of 0 lies outside MoveIt's Panda
  limits (at most -0.0698), so set a valid start pose before planning.

## Measure the result

A `SUCCEEDED` execution result means the controller reached its goal
tolerance on its own state feedback. Read the simulated end-effector pose,
contacts, and the payload's pose from the simulator for the task check, and
record joint tracking error as evidence for gains and tolerances.

## Failures

| Symptom | Check |
|---|---|
| Hardware fails to load | Plugin package installed in the image and named exactly in the URDF |
| `command_interface not found` | Joint names across URDF, controller YAML, and SRDF |
| Controllers load but stay inactive | Spawner output; hardware activated; interface already claimed |
| Arm jumps at activation | Initial state not synced; `initial_value` or first state read |
| Execution aborts on path tolerance | Drive gains in USD, joint limits, time parameterization scaling |
| "No motion plan found" | Start or goal state in collision or out of bounds; IK reachability |
| Plan ignores a table | Collision object missing from the planning scene |
| Joint states jump by 2π | Wrapped continuous joints; unwrap on the hardware side |
| RViz shows the robot, arm never moves | `isaac_joint_commands` has no subscriber in the simulator, or the timeline is stopped |

Use the [bridge guide](isaac-ros2-bridge.md#joint-state-and-joint-commands) for the simulator's joint graph, Isaac Sim [USD articulation](../../isaac-sim-6/references/usd-articulation.md) for drive gains, and [scenario checks](../../scenario-design/SKILL.md#measured-verdicts) to record the task result. Return to [ROS 2](../SKILL.md#references) for where ROS runs and the other guides.
