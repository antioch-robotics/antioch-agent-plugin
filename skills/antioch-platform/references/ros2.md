# ROS 2 across services

The engine bundles in-process ROS 2 Jazzy Python, not a complete `/opt/ros`
development toolchain. Put C++ tools, extra messages, and built workspaces in
the service image. Enable `isaacsim.ros2.bridge` through
`SimulationConfig.extensions` when using the Isaac bridge.

Exec and health checks do not run image entrypoint initialization. Source the
required ROS/workspace environment in those commands. Two engine-backed
services need explicit runner selection in the manifest.

## Discovery and transport

Service names resolve declared routes. Multicast and shared memory do not
cross isolated services. A discovery server finds participants but does not
relay their application data: configure reachable data locators/ports and
verify actual publisher-to-subscriber messages, QoS, and both directions.

`ROS_DISCOVERY_SERVER` selects a Fast DDS endpoint such as `ros:11811`.
`ROS_STATIC_PEERS` lists semicolon-separated peer host addresses, not
`service:discovery-port` values. Variables depend on the selected RMW.
Communicating nodes need compatible domain settings.

See [Jazzy discovery options](https://docs.ros.org/en/jazzy/p/rmw/generated/structrmw__discovery__options__s.html)
and [Fast DDS variables](https://fast-dds.docs.eprosima.com/en/2.14.x/fastdds/env_vars/env_vars.html).

## A point-to-point bridge

A project-configured bridge such as Zenoh can use a named TCP route.
Its image owns ROS setup, bridge mode, security, and topic selection:

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

Configure the laptop bridge for that endpoint using its versioned docs.
Only declared routes are forwarded. Use `--no-stream` for ROS-only service
commands; Isaac WebRTC does not supply an X display for arbitrary RViz apps.

For Nav2, inspect installed launch files and verify map, simulation clock,
TF, sensors, odometry, and control interfaces. Test traffic and controller
outcomes, not just topic discovery. Bake reproducible builds into the image;
exclude generated build/install/log directories from source sync.
