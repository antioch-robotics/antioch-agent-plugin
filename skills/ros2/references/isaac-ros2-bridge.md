# Isaac Sim ROS 2 Bridge

Adapted from NVIDIA [`isaac-sim-ros2-bridge/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/2469084bc328710207c6bc4ede32209082a9286c/skills/isaac-sim-ros2-bridge/SKILL.md) (Apache-2.0). Node types, inputs, and services are checked against the [Isaac Sim v6.0.1 source](https://github.com/isaac-sim/IsaacSim/tree/v6.0.1/source/extensions); that release's node inputs differ from the upstream skill's fleet script, which is not reproduced here.

The bridge publishes and subscribes through OmniGraph action graphs inside
the simulator process. Scene code, startup, and stepping belong to
[Isaac Sim](../../isaac-sim-6/SKILL.md); ROS nodes outside the simulator
belong in a ROS service image.

## Enable the extensions

```python
import antioch

antioch.start_simulation(antioch.SimulationConfig(extensions=["isaacsim.ros2.bridge", "isaacsim.ros2.sim_control"]))
```

| Extension | Provides |
|---|---|
| `isaacsim.ros2.bridge` | The bridge and its node library; enables `isaacsim.ros2.core` and `isaacsim.ros2.nodes` |
| `isaacsim.ros2.sim_control` | `simulation_interfaces` services and the `simulate_steps` action |
| `isaacsim.ros2.urdf` | Import a robot from a running node's `robot_description` |
| `isaacsim.ros2.tf_viewer` | TF tree view in the Isaac UI |

The engine image sets `ROS_DISTRO`, `RMW_IMPLEMENTATION`, `AMENT_PREFIX_PATH`,
and the library path before Kit starts. The dynamic loader reads the library
path only at process start, so a changed ROS environment needs a new process
or a new image, never an `export` inside a running simulator. Set
`ROS_DOMAIN_ID` and discovery variables on the service in the manifest.

## Build graphs

Use the `isaacsim.ros2.bridge.ROS2*` node types; the old
`omni.isaac.ros2_bridge.*` names do not load. Create graphs after the stage
opens and before the timeline plays. `OnPlaybackTick` drives them once per
app update while the timeline plays: nothing publishes while stopped, and a
physics-only step without an app update publishes nothing. Use
`isaacsim.core.nodes.OnPhysicsStep` to publish per physics step. Stamp every
message from `isaacsim.core.nodes.IsaacReadSimulationTime`.

This graph publishes `/clock`, a ground-truth `odom` → base transform with
odometry, the sensor frames under the base, and subscribes to `cmd_vel`.
`base_prim` is the robot's base rigid body and `sensor_prims` the sensor or
mount prims fixed to it:

```python
def build_mobile_base_graph(base_prim: str, sensor_prims: list[str], graph_path: str = "/World/ROS2Bridge") -> None:
    import omni.graph.core as og
    import usdrt.Sdf

    # The transform tree names the base after its prim; odometry must match
    base_frame = base_prim.rsplit("/", 1)[-1]
    keys = og.Controller.Keys
    og.Controller.edit(
        {"graph_path": graph_path, "evaluator_name": "execution"},
        {
            keys.CREATE_NODES: [
                ("Tick", "omni.graph.action.OnPlaybackTick"),
                ("SimTime", "isaacsim.core.nodes.IsaacReadSimulationTime"),
                ("Context", "isaacsim.ros2.bridge.ROS2Context"),
                ("Clock", "isaacsim.ros2.bridge.ROS2PublishClock"),
                ("Odometry", "isaacsim.core.nodes.IsaacComputeOdometry"),
                ("PublishOdometry", "isaacsim.ros2.bridge.ROS2PublishOdometry"),
                ("OdomTF", "isaacsim.ros2.bridge.ROS2PublishRawTransformTree"),
                ("LinkTree", "isaacsim.core.nodes.IsaacComputeTransformTree"),
                ("LinkTF", "isaacsim.ros2.bridge.ROS2PublishTransformTree"),
                ("CmdVel", "isaacsim.ros2.bridge.ROS2SubscribeTwist"),
            ],
            keys.CONNECT: [
                ("Tick.outputs:tick", "Clock.inputs:execIn"),
                ("Tick.outputs:tick", "Odometry.inputs:execIn"),
                ("Tick.outputs:tick", "LinkTree.inputs:execIn"),
                ("Tick.outputs:tick", "CmdVel.inputs:execIn"),
                ("Odometry.outputs:execOut", "PublishOdometry.inputs:execIn"),
                ("Odometry.outputs:execOut", "OdomTF.inputs:execIn"),
                ("Odometry.outputs:position", "PublishOdometry.inputs:position"),
                ("Odometry.outputs:orientation", "PublishOdometry.inputs:orientation"),
                ("Odometry.outputs:linearVelocity", "PublishOdometry.inputs:linearVelocity"),
                ("Odometry.outputs:angularVelocity", "PublishOdometry.inputs:angularVelocity"),
                ("Odometry.outputs:position", "OdomTF.inputs:translation"),
                ("Odometry.outputs:orientation", "OdomTF.inputs:rotation"),
                ("LinkTree.outputs:execOut", "LinkTF.inputs:execIn"),
                ("LinkTree.outputs:parentFrames", "LinkTF.inputs:parentFrames"),
                ("LinkTree.outputs:childFrames", "LinkTF.inputs:childFrames"),
                ("LinkTree.outputs:translations", "LinkTF.inputs:translations"),
                ("LinkTree.outputs:orientations", "LinkTF.inputs:orientations"),
                *(("Context.outputs:context", f"{name}.inputs:context") for name in ("Clock", "PublishOdometry", "OdomTF", "LinkTF", "CmdVel")),
                *(("SimTime.outputs:simulationTime", f"{name}.inputs:timeStamp") for name in ("Clock", "PublishOdometry", "OdomTF", "LinkTF")),
            ],
            keys.SET_VALUES: [
                ("Odometry.inputs:chassisPrim", [usdrt.Sdf.Path(base_prim)]),
                ("PublishOdometry.inputs:chassisFrameId", base_frame),
                ("PublishOdometry.inputs:publishRawVelocities", True),
                ("OdomTF.inputs:childFrameId", base_frame),
                ("LinkTree.inputs:parentPrim", [usdrt.Sdf.Path(base_prim)]),
                ("LinkTree.inputs:targetPrims", [usdrt.Sdf.Path(path) for path in sensor_prims]),
            ],
        },
    )
```

`CmdVel` outputs linear and angular velocity; connect them to the robot's
drive, such as a differential controller feeding an articulation
controller. A published command does nothing until something consumes it.

`IsaacComputeOdometry` outputs `linearVelocity` and `angularVelocity` in the
robot's frame, so `publishRawVelocities` is on: left off, `ROS2PublishOdometry`
treats them as world velocities and rotates them a second time.

`IsaacComputeOdometry` reports motion relative to the robot's start, so its
`odom` frame starts at the start pose. It is ground truth, not wheel
odometry: a stack whose localization copes with wheel slip needs a noisy or
wheel-derived source. Publish `map` → `odom` from exactly one place, such as
AMCL, SLAM, or a static ground-truth transform, never two.

Exactly one publisher owns `odom` → base: here `OdomTF`. So `LinkTree`
takes the base as `parentPrim` and only prims below it as targets; a blank
`parentPrim` publishes `world` → each target, and an articulation-root target
expands to every link including the base, giving the base a second parent.
Moving links such as wheels or an arm come from `robot_state_publisher` fed
by `ROS2PublishJointState`, never from both sources.

Frame IDs from `IsaacComputeTransformTree` are the prim's
`isaac:nameOverride`, or else its leaf name, so the base frame is
`chassis_link` for a prim `/World/Carter/chassis_link`. Sensor helpers and
the graphs in NVIDIA's sample robots set `frameId` explicitly, and it need
not match the prim (Carter's lidar publishes `XT_32_10hz` from a
`XT_32_10Hz` prim); read the published names from `/tf`, not the stage.
Keep that name, or author `isaac:nameOverride = "base_link"` on the base and
pass `base_link` as `chassisFrameId` and `childFrameId`; either way the name
must match the URDF and every Nav2 `robot_base_frame`.

## Nodes

Publishers: `ROS2PublishClock`, `ROS2PublishJointState`,
`ROS2PublishOdometry`, `ROS2PublishTransformTree`,
`ROS2PublishRawTransformTree`, `ROS2PublishLaserScan`,
`ROS2PublishPointCloud`, `ROS2PublishImage`, `ROS2PublishCompressedImage`,
`ROS2PublishCameraInfo`, `ROS2PublishImu`, `ROS2PublishBbox2D`,
`ROS2PublishBbox3D`, `ROS2PublishSemanticLabels`, `ROS2PublishObjectIdMap`,
`ROS2PublishAckermann`, and the generic `ROS2Publisher`.

Subscribers: `ROS2SubscribeTwist` (`geometry_msgs/Twist`),
`ROS2SubscribeJointState`, `ROS2SubscribeAckermann`, `ROS2SubscribeClock`,
`ROS2SubscribeTransformTree`, and the generic `ROS2Subscriber`. Services use
`ROS2ServiceClient`, `ROS2ServiceServerRequest`, `ROS2ServiceServerResponse`,
and `ROS2ServicePrim`.

Helpers build sensor pipelines from a render product: `ROS2CameraHelper`
(`type` of `rgb`, `depth`, `depth_pcl`, segmentation, or bounding boxes),
`ROS2CameraInfoHelper`, and `ROS2RtxLidarHelper` (`laser_scan` or
`point_cloud`). A camera or RTX lidar publishes only with a render product
attached; see Isaac Sim [sensors](../../isaac-sim-6/references/isaac-sim-sensor.md).
A 2D scan from an RTX lidar prim, added to the graph above and verified at
10 Hz of simulation time on a Nova Carter:

```python
def add_lidar_scan(graph_path: str, lidar_prim: str, frame_id: str, topic: str = "scan") -> None:
    import omni.graph.core as og
    import usdrt.Sdf

    keys = og.Controller.Keys
    og.Controller.edit(
        graph_path,
        {
            keys.CREATE_NODES: [
                ("RunOnce", "isaacsim.core.nodes.OgnIsaacRunOneSimulationFrame"),
                ("LidarRP", "isaacsim.core.nodes.IsaacCreateRenderProduct"),
                ("Scan", "isaacsim.ros2.bridge.ROS2RtxLidarHelper"),
            ],
            keys.CONNECT: [
                ("Tick.outputs:tick", "RunOnce.inputs:execIn"),
                ("RunOnce.outputs:step", "LidarRP.inputs:execIn"),
                ("LidarRP.outputs:execOut", "Scan.inputs:execIn"),
                ("LidarRP.outputs:renderProductPath", "Scan.inputs:renderProductPath"),
                ("Context.outputs:context", "Scan.inputs:context"),
            ],
            keys.SET_VALUES: [
                ("LidarRP.inputs:cameraPrim", [usdrt.Sdf.Path(lidar_prim)]),
                ("Scan.inputs:type", "laser_scan"),
                ("Scan.inputs:topicName", topic),
                ("Scan.inputs:frameId", frame_id),
            ],
        },
    )
```

The render product is created once, on the first tick. The published
`LaserScan` marks beams with no return as `-1.0`, not `+inf` as REP 117
expects; a range check must reject them, and a Nav2 costmap treats them as
invalid rather than clearing free space along them. The helper logs that
`fullScan` is deprecated whether or not it is set; the warning is harmless.

Every publisher and subscriber takes `nodeNamespace`, `topicName`,
`queueSize`, and `qosProfile`. An empty `qosProfile` means reliable,
volatile, keep-last depth 10; build others with `ROS2QoSProfile`
(`createProfile` of `sensorData`, `services`, or `custom`). A reliable
publisher also serves best-effort subscribers such as Nav2's scan input.

## Joint state and joint commands

`ROS2PublishJointState` takes its names, positions, velocities, efforts,
DOF types, and stage units from `isaacsim.sensors.physics.IsaacReadJointState`
pointed at the articulation. `ROS2SubscribeJointState` feeds
`isaacsim.core.nodes.IsaacArticulationController`. NVIDIA's MoveIt sample
uses the topics `isaac_joint_states` and `isaac_joint_commands`; see
[MoveIt 2 and ros2_control](moveit2-ros2-control.md) for the controller side.

Joint names must match the URDF the ROS stack loads. Revolute positions are
radians and prismatic positions meters when the stage is in meters.

## Frames and quaternions

OmniGraph `quatd` values are IJKR, `(x, y, z, w)`, so identity is
`(0, 0, 0, 1)`. `IsaacComputeTransformTree` outputs `(x, y, z, w)` too.
Isaac core Python APIs return WXYZ. Frame IDs come from
`isaac:nameOverride` or the prim's leaf name unless a node takes an explicit
frame input; renaming or moving prims at runtime changes the published tree.

## Multiple robots

Give each robot its own graph with `nodeNamespace` set to the robot name, and
frame IDs prefixed to match its Nav2 or `robot_state_publisher`
configuration, such as `robot1/odom` and `robot1/base_link`. Publish one
shared `/clock`. Localization for each robot publishes `map` →
`robot1/odom`.

## Simulation control

`isaacsim.ros2.sim_control` implements the `simulation_interfaces` standard
with root-namespace services: `get_simulation_state`,
`set_simulation_state`, `step_simulation`, `reset_simulation`,
`get_entities`, `get_entity_state`, `get_entities_states`,
`set_entity_state`, `get_entity_bounds`, `get_entity_info`, `spawn_entity`,
`spawn_entities`, `delete_entity`, `get_spawnables`, `load_world`,
`unload_world`, `get_current_world`, `get_available_worlds`, and
`get_simulator_features`, plus the `simulate_steps` action. `StepSimulation`
and `simulate_steps` refuse while playing; from pause they play the timeline
for the requested number of app updates, then pause again. An app update is
not necessarily one physics step, a request for one step is unsupported, and
both need the simulator's app loop to keep running to finish. Read
simulation time before and after when an exact step count matters.

The extension initializes `rclpy` in the simulator process and spins its own
executor thread. Check `rclpy.ok()` before `rclpy.init()` and do not call
`rclpy.shutdown()` in that process. The scenario or script that steps the
simulator still owns its loop: an external stepper and the process's own
`world.step()` calls interleave.

## Failures

| Symptom | Check |
|---|---|
| No topics | Timeline playing, extension in `SimulationConfig.extensions`, matching `ROS_DOMAIN_ID` and discovery settings |
| Topics listed, no messages | Subscriber QoS against the publisher, then `ros2 topic hz` |
| Messages stop when stepping without rendering | `OnPlaybackTick` needs app updates; use `OnPhysicsStep` |
| Stamps in wall time or zero | `timeStamp` wired to `IsaacReadSimulationTime` |
| Consumer times out on TF | `/clock` advancing and `use_sim_time: true` on the consumer |
| A frame has two parents | `LinkTree` `parentPrim` set to the base, the base not among its targets, and one source for moving links |
| Robot ignores `cmd_vel` | Twist subscriber wired to the drive; the sender publishes `Twist`, not `TwistStamped` |
| Camera topic silent | Render product attached to the camera helper |
| Undefined symbol or missing type support at startup | Engine ROS environment replaced or a custom interface not built for its Jazzy Python |

NVIDIA's sample scenes under `/Isaac/Samples/ROS2/Scenario/` in the Isaac
asset root, such as `carter_warehouse_navigation.usd`, come with bridge
graphs; load them as described in
[assets](../../antioch-platform/references/assets.md). Their graphs publish
what the sample needs, not what Nav2 expects: Carter's XT32 lidar publishes
a 3D `/point_cloud` and no `/scan`, and in one headless run it delivered no
messages at all. Subscribe and count messages before building on a sample
sensor; add a 2D scan with `add_lidar_scan` on a 2D lidar prim when needed.

Use [tf2 and time](tf2-and-time.md) to check the frames and clock this graph publishes, [nodes and QoS](nodes-and-qos.md) when a topic is listed but silent, and [scenario checks](../../scenario-design/SKILL.md#measured-verdicts) to record what the robot did. Return to [ROS 2](../SKILL.md#references) for where ROS runs and the other guides.
