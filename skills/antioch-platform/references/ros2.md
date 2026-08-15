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
down`. Queued runs use their saved service images, project files, and inputs
without development watch rules.

## Run the Nova Carter warehouse stack

The documented warehouse launch file is absent from the installed
`carter_navigation` package share directory on Jazzy. The package does ship
the warehouse map and the Nav2 parameter file, so start Nav2 directly after
you build and source the workspace:

```bash
antioch services exec ros bash -lc 'source /opt/ros/jazzy/setup.bash && source /workspace/project/ros_ws/install/setup.bash && ros2 launch nav2_bringup bringup_launch.py map:=/workspace/project/ros_ws/install/carter_navigation/share/carter_navigation/maps/carter_warehouse_navigation.yaml params_file:=/workspace/project/ros_ws/install/carter_navigation/share/carter_navigation/params/carter_navigation_params.yaml use_sim_time:=True'
```

This uses the files installed by `carter_navigation`. The path is not the
source path. Keep the same `ROS_DOMAIN_ID` in the `sim` and `ros` services.

RViz exits on a headless machine because there is no display. This is
expected. It does not mean that Nav2 or Antioch failed. Use an interactive
Antioch Isaac stream to see the simulator scene. For example:

```bash
antioch run --stream src/main.py
```

Open the machine livestream in Mission Control. `antioch services exec` does
not claim the stream. A Rerun view is available only when the running scenario
or session logs Rerun data; the Nav2 process alone does not create that data.

Nav2 also needs a TF tree that contains `map`, `odom`, and `base_link`. The
bringup command does not create that tree. Until the simulator or robot driver
publishes it, messages such as `frame "map" does not exist` and
`Timed out waiting for transform from base_link to map` are expected. They
describe missing robot TF, not a missing map file or an Antioch failure.

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
