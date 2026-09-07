# ROS 2 and autonomy services

Use this when connecting ROS nodes, enabling the Isaac bridge, or adding a
supporting autonomy service. Preserve the user's graph, middleware, and topic
contract; a discovery workaround is not proof of message delivery.

The engine bundles an in-process ROS 2 Jazzy Python stack. That does not
promise a complete `/opt/ros` development toolchain. Put C++ tools, extra
messages, and a built workspace in the service image when needed.
Opt into `isaacsim.ros2.bridge` through `SimulationConfig.extensions` for
Isaac bridge work.

## Process and image boundaries

A simulator service can have any name; `sim` is the generated default.
Use its generated, versioned Dockerfile through `build: .` or an explicitly
versioned engine image. Untagged engine image references are not a runnable
example.

Supporting services have their own command, dependencies, resources, and
named routes. Two engine-backed services need explicit runner selection with
`x-antioch: {runner: true}` on the intended simulator.

Healthchecks and `antioch services exec` do not run an image's entrypoint
initialization first. Source the ROS/workspace environment in the command
when that image requires it. A healthcheck that merely prints "ready" or
lists an empty topic graph does not prove the required nodes are serving.

## Discovery is not data transport

Service names resolve through Antioch's declared service routes. Do not rely
on multicast or shared-memory discovery across isolated services.

`ROS_DISCOVERY_SERVER` selects a Fast DDS discovery server, for example a
configured `ros:11811` endpoint. `ROS_STATIC_PEERS` instead lists peer host
addresses separated by semicolons; it is not an equivalent
`service:discovery-port` setting. Supported variables depend on the selected
RMW implementation.
See [ROS discovery options](https://docs.ros.org/en/jazzy/p/rmw/generated/structrmw__discovery__options__s.html)
and [Fast DDS environment variables](https://fast-dds.docs.eprosima.com/en/2.14.x/fastdds/env_vars/env_vars.html).
The [Jazzy Fast DDS participant implementation](https://github.com/ros2/rmw_fastrtps/blob/jazzy/rmw_fastrtps_shared_cpp/src/participant.cpp)
resolves each static peer as a DNS/IP host and lets Fast DDS choose participant
ports; do not append a discovery-server port to that host name.

A discovery server does not relay application data. DDS participants still
need reachable data locators and ports. Declaring one discovery port alone
does not make a cross-service ROS graph work. Validate actual publisher to
subscriber traffic, including QoS and both directions, rather than treating
`ros2 topic list` as acceptance.

Use the same domain for nodes intended to communicate where the middleware's
domain semantics apply; isolate independent graphs deliberately. Do not give
every communicating node a different domain ID.

## A declared bridge route

A point-to-point bridge such as a project-configured Zenoh bridge can use one
declared TCP route. Its image owns the bridge mode, ROS setup, endpoints,
security, and topic allow-list. This manifest fragment only declares transport:

```yaml
services:
  bridge:
    image: registry.example.com/robot/zenoh-bridge@sha256:<digest>
    ports:
      - name: ros2-zenoh
        port: 7447
        protocol: tcp
        direction: client-to-service
```

Replace the image placeholder with the project's actual published image.
For a laptop bridge endpoint, start/select the session, bind the route, and
keep the foreground forwarder alive:

```bash
antioch session start
antioch services ports --bind bridge.ros2-zenoh=127.0.0.1:7447
antioch services ports --serve
```

Configure the laptop endpoint for `127.0.0.1:7447` using that bridge's release
documentation. Only declared client-to-service routes are forwarded. Stop
the owned forwarder when the task no longer needs it.

## Run and validate

For a service whose image contains the required ROS installation:

```bash
antioch services exec --no-stream --service ros -- bash -lc 'source /opt/ros/jazzy/setup.bash && ros2 run demo_nodes_cpp talker'
```

Use `--no-stream` for ROS-only commands so they do not claim the one Isaac
GUI stream. Isaac WebRTC does not automatically provide an X display for
arbitrary RViz processes.

Retrieve the installed launch files and package share paths before composing
Nav2 bringup. Do not assume a private workspace, historical Carter launch
file, or hard-coded installed path exists. Nav2 also needs the correct map,
simulation clock, TF tree, sensors, odometry, and control interfaces.

Record task-specific readiness, message flow, timestamps, TF consistency, and
controller outcome. Put a reproducible build in the image; an interactive
`colcon build` is temporary development state. Keep generated build/install/log
directories out of source sync, and retain logs and measurements as run
artifacts when durable evidence is requested.
