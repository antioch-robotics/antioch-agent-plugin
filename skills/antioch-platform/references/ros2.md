# ROS 2 across services

The engine bundles in-process ROS 2 Jazzy Python, not a complete `/opt/ros`
development toolchain. Put C++ tools, extra messages, and built workspaces in
the service image. Enable `isaacsim.ros2.bridge` through
`SimulationConfig.extensions` when using the Isaac bridge.

Exec and health checks do not run image entrypoint initialization, so source
the required ROS/workspace environment in those commands. With two
engine-backed services, a scenario names its own with `service=` in the
decorator and a command with `--service`.

## Discovery and transport

Service names resolve declared routes. Multicast and shared memory do not
cross isolated services. A discovery server finds participants but does not
relay their application data: configure reachable data locators/ports and
verify actual publisher-to-subscriber messages, QoS, and both directions.
Multicast between services can happen to work on some cells; the CLI still
warns, because it is not guaranteed. Configure a discovery server:

```yaml
services:
  sim:
    environment:
      ROS_DISCOVERY_SERVER: "ros:11811"
      FASTDDS_BUILTIN_TRANSPORTS: UDPv4
  ros:
    command: ["bash", "-c", "source /opt/ros/jazzy/setup.bash && exec fastdds discovery --server-id 0 --udp-address 0.0.0.0 --udp-port 11811"]
    ports:
      - name: discovery
        port: 11811
        protocol: udp
        direction: client-to-service
    environment:
      ROS_DISCOVERY_SERVER: "ros:11811"
      FASTDDS_BUILTIN_TRANSPORTS: UDPv4
```

Run the ROS stack itself with `antioch service exec --service ros` or a
second process in that service; the discovery server is the service command.

`ROS_DISCOVERY_SERVER` selects a Fast DDS endpoint such as `ros:11811`.
`ROS_STATIC_PEERS` lists semicolon-separated peer host addresses, not
`service:discovery-port` values. Variables depend on the selected RMW, and
communicating nodes need compatible domain settings. See
[Jazzy discovery options](https://docs.ros.org/en/jazzy/p/rmw/generated/structrmw__discovery__options__s.html)
and [Fast DDS variables](https://fast-dds.docs.eprosima.com/en/2.14.x/fastdds/env_vars/env_vars.html).

## A point-to-point bridge

A project-configured bridge such as Zenoh can use a named TCP route. Its
image owns ROS setup, bridge mode, security, and topic selection:

```yaml
services:
  bridge:
    ports:
      - name: ros2-zenoh
        port: 7447
        protocol: tcp
        direction: client-to-service
```

For a service named `bridge`, bind the laptop endpoint and keep serving:

```bash
antioch service ports --bind bridge.ros2-zenoh=127.0.0.1:7447
antioch service ports --serve
```

Only declared routes are forwarded. Isaac's WebRTC stream does not supply
an X display for arbitrary RViz apps.

Bake reproducible builds into the image and exclude generated
build/install/log directories from source sync.

Continue with [manifest routes](manifest.md#routes) and [session forwarding](sessions.md#named-routes) to connect services, [ROS 2](../../ros2/SKILL.md) for the Isaac bridge, simulation time, tf2, QoS, Nav2, MoveIt 2, and ros2_control inside the graph, and [scenario checks](../../scenario-design/SKILL.md#measured-verdicts) to measure the full loop. [Antioch platform](../SKILL.md#capability-guides) is the task index.
