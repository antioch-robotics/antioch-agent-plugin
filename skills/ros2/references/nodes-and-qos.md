# Nodes, Executors, QoS, and Discovery

Adapted from [`references/communication.md`](https://github.com/dbwls99706/ros2-engineering-skills/blob/61870e4b94098d4d4f4b1f640936f9a3f7721304/references/communication.md), [`references/nodes-executors.md`](https://github.com/dbwls99706/ros2-engineering-skills/blob/61870e4b94098d4d4f4b1f640936f9a3f7721304/references/nodes-executors.md), and [`references/launch-system.md`](https://github.com/dbwls99706/ros2-engineering-skills/blob/61870e4b94098d4d4f4b1f640936f9a3f7721304/references/launch-system.md) in ros2-engineering-skills (Apache-2.0). Discovery variables are checked against Fast DDS 2.14, the Jazzy default middleware.

Routes and discovery between Antioch services belong to the platform
[ROS 2 guide](../../antioch-platform/references/ros2.md); this reference owns
what happens inside a node and on the wire.

## Choose the interface

| Need | Use |
|---|---|
| Continuous data: sensors, state, commands | Topic |
| Short request and reply that returns promptly | Service |
| Long task with feedback and cancel: navigate, move an arm | Action |
| Configuration read at start or changed rarely | Parameter |

## QoS

A subscription connects only to publishers whose offered QoS meets what it
requests. A mismatch shows up as missing traffic: both ends exist and
nothing flows. rclpy logs an incompatible-QoS warning when the middleware
reports it, so read the node's log as well as the endpoints.

| Publisher offers | Subscriber requests | Connects |
|---|---|---|
| Reliable | Reliable or best effort | Yes |
| Best effort | Best effort | Yes |
| Best effort | Reliable | **No** |
| Transient local | Transient local or volatile | Yes |
| Volatile | Transient local | **No** |

- Sensor streams: `rclpy.qos.qos_profile_sensor_data` (best effort,
  volatile, keep last 5). Nav2 and most perception nodes subscribe this way.
- Maps, `robot_description`, and `/tf_static`: reliable, transient local,
  depth 1, so a late subscriber still receives the last message.
- Commands: reliable with a small depth; a deep queue replays stale commands.

`ros2 topic info -v TOPIC` prints every endpoint's QoS. `ros2 topic echo`
adapts its QoS to the publishers, so its output does not prove your
subscriber matches; pass `--qos-reliability` and `--qos-durability` to test
your profile.

## Executors and callbacks

`rclpy.spin(node)` runs a single-threaded executor: one callback at a time.

- Never wait synchronously for a service or action result inside a callback
  on a single-threaded executor; the reply's callback cannot run. Use
  `call_async` with a done callback, or a `MultiThreadedExecutor` with the
  client in a different callback group from the calling callback, or both in
  a `ReentrantCallbackGroup`. A `MultiThreadedExecutor` alone still deadlocks:
  every entity shares the node's default mutually exclusive group.
- `spin_until_future_complete` belongs outside callbacks.
- A slow callback delays every other callback on its executor, including
  `/clock` and TF subscriptions. Move heavy work off the executor.
- Inside the simulator, spin with `spin_once(node, timeout_sec=0.0)` between
  steps or run an executor on a daemon thread; see the
  [skill](../SKILL.md#python-inside-the-engine).

## Lifecycle nodes

Nav2 servers and many drivers are managed nodes: unconfigured → inactive →
active. A lifecycle manager drives the transitions. An inactive node exists
in `ros2 node list` but does not process data. In a ROS image,
`ros2 lifecycle get /NODE` shows its state; behind a discovery server, where
it can report "Node not found", call the node's service instead:
`ros2 service call /NODE/get_state lifecycle_msgs/srv/GetState`.

ros2_control hardware components follow the same states but are not nodes:
the controller manager owns them. Read them with
`ros2 control list_hardware_components` and change them with
`ros2 control set_hardware_component_state`. Inactive hardware reads state;
its command interfaces are nominally unavailable, but Jazzy still writes
commands and leaves the component to ignore them. Controllers load and
activate through the controller manager and its spawner.

## Parameters

```yaml
controller_server:
  ros__parameters:
    use_sim_time: true
/**:
  ros__parameters:
    use_sim_time: true
```

The node name keys its block and `/**` matches every node. A launch file's
`parameters=[...]` overrides the YAML it follows. The source file is not
proof of the running value; read the live one with `ros2 param get /NODE
NAME` in a ROS image.

## Launch

Run launch files in a ROS service image with its workspace sourced:
`ros2 launch PACKAGE FILE.launch.py use_sim_time:=true`. The engine has the
`launch` Python packages but no `ros2 launch` verb. Launch substitutions are
evaluated at launch time, so print the resolved parameters or read them live
when a value matters. Exec and health checks skip image entrypoints; source
`/opt/ros/$ROS_DISTRO/setup.bash` and the workspace's `install/setup.bash` in the
command itself.

## Discovery and middleware

Every node in one graph needs the same `ROS_DOMAIN_ID` and a compatible
`RMW_IMPLEMENTATION`; the engine and `ros:jazzy` both default to
`rmw_fastrtps_cpp`. Between services, multicast does not cross; a Fast DDS
discovery server set through `ROS_DISCOVERY_SERVER` replaces it.

- With a discovery server, the `ros2` CLI sees only what its own node
  needs. Set `ROS_SUPER_CLIENT=TRUE` for introspection and restart the daemon
  with `ros2 daemon stop`; `--no-daemon` works only on verbs that accept it,
  and `ros2 topic hz` does not.
- Discovery succeeding while data never arrives often means Fast DDS chose
  shared memory between processes that do not share it. Force UDP with
  `FASTDDS_BUILTIN_TRANSPORTS=UDPv4` on both ends.
- Large messages such as images and point clouds fragment over UDP; test the
  real message size and rate, not a string topic.
- Zenoh (`rmw_zenoh_cpp`) needs its router reachable from every node and a
  `RMW_IMPLEMENTATION` change everywhere; it does not interoperate with DDS
  nodes.

## Custom interfaces

Generate messages in a ROS package built in the ROS service image, and keep
the nodes that use them there. The engine loads only the interfaces it
ships unless type support is built for its bundled Jazzy Python 3.12. A
message's definition must be identical on both ends; a mismatch under the
same type name fails to connect or delivers garbage.

## Failures

| Symptom | Check |
|---|---|
| Publisher and subscriber listed, no data | QoS with `ros2 topic info -v`, then shared memory and UDP reachability |
| Topics from another service absent | Domain ID, RMW, discovery server address, `ROS_SUPER_CLIENT` for the CLI |
| Service call never returns | Synchronous wait inside a callback; server's executor busy |
| Action feedback missing | Client node not being spun |
| "Failed to find type support" | Interface package not built or not sourced in that process |
| Node present but idle | Lifecycle state inactive |
| Parameter change ignored | Wrong node key in YAML, or a launch override after it |
| Old commands replayed at start | Deep reliable queue or transient-local command topic |

Use [manifest routes](../../antioch-platform/references/manifest.md#routes) to connect services, the [bridge guide](isaac-ros2-bridge.md) for the simulator's QoS, and [tf2 and time](tf2-and-time.md) when data arrives but frames or stamps are wrong. Return to [ROS 2](../SKILL.md#references) for where ROS runs and the other guides.
