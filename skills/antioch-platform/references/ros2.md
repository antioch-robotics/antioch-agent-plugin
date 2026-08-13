# ROS 2 and autonomy services

The Isaac Sim base image includes the in-process ROS 2 Jazzy Python stack, so
`import rclpy` works in a scenario, a native run, or a Jupyter kernel. The base
does not promise the full `/opt/ros` command-line toolchain. Add C++ tooling,
`colcon`, or extra message packages in the sim Dockerfile when the project
needs them.

## Declare the graph

Autonomy containers are ordinary entries under `services`; `sim` is the
reserved simulation service, and no other service may use an engine image:

```yaml
services:
  sim:
    build: .
    environment:
      ROS_DOMAIN_ID: "7"
    depends_on:
      ros: {condition: service_healthy}
    watch:
      - action: sync
        path: .
        target: /workspace/project
  ros:
    build: ./ros
    healthcheck:
      test: ["CMD", "bash", "-c", "source /opt/ros/jazzy/setup.bash && ros2 topic list"]
      interval: 2s
      timeout: 5s
      retries: 15
```

Healthchecks and `antioch services exec` run without the image entrypoint that
sources `/opt/ros/jazzy/setup.bash`, so wrap every ROS command in `bash -c`
with an explicit `source`, as above — a bare `ros2` probe fails with
"executable file not found" on stock ROS images.

Host networking and IPC default on every service, so DDS discovery works
through localhost without a discovery server or a peer list. A service can
opt out with an explicit supported value — `network_mode: none` or `bridge`,
`ipc: private`, or a `service:NAME` namespace reference for deliberate
sharing; `manifest.md` owns the complete value sets. A service needed only by
selected runs can carry an auxiliary `profiles` value. Use `antioch services up --watch` for the live graph; it is
foreground, and Ctrl-C leaves containers running until `antioch services
down`. Queued runs use a digest-pinned environment without development watch
rules.

## Build a C++ workspace

Put the ROS apt repository, compiler, `ros-dev-tools`, and required message
packages in the Dockerfile. Then use the running sim service for an interactive
build:

```bash
antioch services exec sim bash -lc 'source /opt/ros/jazzy/setup.bash && cd ros_ws && colcon build'
```

The workspace is under `/workspace/project`, so build products are visible to
later commands in the live service. Keep generated `build`, `install`, and
`log` directories out of the build context with `.dockerignore` and the
`watch.ignore` list. For reproducible scenarios, put the toolchain in the
Dockerfile and let Antioch build it; do not rely on an unrecorded
interactive shell.

## Code-level escape hatch

`antioch.container()` remains useful for a short-lived helper that does not
belong in the service graph. It starts and tears down a container inside the
current run and can use the same host network:

```python
import antioch

with antioch.container("ros:jazzy", ready=antioch.tcp_ready(9090)):
    ...
```

Use `antioch.command_ready([...])` when readiness is a command rather than a
port. Give parallel cases distinct container names, ROS domains, and ports;
host networking makes those values machine-wide. Keep `reuse=True` only when
cases are intentionally serialized and the adopted service is known to be the
right one.

## Common ROS boundaries

- Keep `ROS_DOMAIN_ID` and every bound port unique when cases can share a
  machine.
- Gate on a service-specific readiness file or command when a shared port could
  belong to a stale process.
- Use `antioch services exec` or a scenario when a launch must be timed, streamed, or
  recorded; `antioch services ssh` is a human shell and has no process record.
- Put durable logs, metrics, and recordings in scenario artifacts. The output
  directory is assignment scratch and is retrieved explicitly before release.
