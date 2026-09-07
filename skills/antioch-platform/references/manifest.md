# Author `antioch.yaml`

`antioch.yaml` is the project manifest. It declares project identity,
scenario discovery, suites, and a service graph. It does not contain cloud
infrastructure configuration. Its service syntax is inspired by
[Docker Compose](https://docs.docker.com/compose/compose-file/).

Start from the file produced by `antioch init`, then make the smallest service
change that supports the workflow.

## Start with one simulation service

```yaml
id: warehouse-sim-0123456789abcdef
name: warehouse-sim
scenario_paths: ["src/scenarios.py"]

services:
  sim:
    build:
      context: .
      dockerfile: Dockerfile
    resources:
      gpu: rtx-pro-6000
    watch:
      - action: sync
        path: .
        target: /workspace/project
      - action: rebuild
        path: Dockerfile
```

The manifest needs at least one service. The simulator is a role of the image,
not a name: the service whose image is an Antioch engine image, or is built
`FROM` one, runs native Isaac scripts, scenarios, suites, and Jupyter.
`antioch init` calls it `sim`. A service-only project has no simulator. A
service names the GPU class in `resources.gpu`; an engine service with no
class gets its engine's recommended class, and no name implies one.

The generated Dockerfile starts from a versioned Antioch engine image. Every
service declares exactly one published `image` or local `build`.

A supporting service may also name the same `antioch-engine/<engine>:<sdk-version>`
image when it needs the bundled ROS 2 stack. Set an explicit command for that
service, and mark the simulator with `x-antioch: {runner: true}`, because two
engine-built services are otherwise ambiguous. A supporting service runs on
the same machine and sees that GPU without a line of its own.

## Add supporting services

A supporting service declares one published image or local build, then its
command, environment, working directory, dependencies, healthcheck, profiles,
resources, restart policy, ports, and watch rules as needed. Unknown fields
are rejected.

For example, this assumes the project's autonomy image serves its control API
on port 8080:

```yaml
services:
  sim:
    build: .
    resources:
      gpu: rtx-pro-6000
  autonomy:
    image: registry.example.com/robot/autonomy:release
    command: ["python", "-m", "robot.main"]
    ports:
      - name: control
        port: 8080
        protocol: tcp
    working_dir: /app
    depends_on:
      sim:
        condition: service_started
    resources:
      cpu: "2"
      memory: 4Gi
    restart: on-failure
```

Service names are network names for declared routes: the simulator can
reach this API at `autonomy:8080`. Process startup is not application
readiness; use a real service-specific healthcheck when the caller needs
a ready dependency. Do not use a command that always succeeds as a
healthcheck.

ROS discovery and data traffic have additional requirements. See
`references/ros2.md`; one declared discovery port is not a complete DDS
transport configuration.

## Gate service startup

`depends_on` names another service and one condition:

- `service_started` waits for the dependency process to start. It is the
  default.
- `service_healthy` waits for the dependency health check.
- `service_completed_successfully` waits for a one-shot dependency to exit
  successfully.

Dependencies must exist, be active in the selected profiles, and form no
cycle.

Health checks accept a shell string or an argument list. Antioch lowers them to
application startup and readiness probes; it never uses them as a liveness
kill loop. Timing values accept duration strings such as `500ms`, `2s`, or `1m`.

## Request resources and placement

A session is one machine. One service declares the GPU class in
`resources.gpu`; that puts the whole session on a machine of that class, and
every service in the session sees every GPU of that machine. A project
declares one class: another service may repeat it, and a different class is
refused naming both services. Antioch does not use MIG, MPS, time-slicing,
fractional devices, or cross-user GPU sharing. A service built `FROM` an
Antioch engine image with no `resources.gpu` gets its engine's recommended
class; a class that engine cannot run is refused naming the fix. A CUDA-only
project sets the class on any one service; a project with no engine service
and no `resources.gpu` runs on CPU-only capacity.

`with` places services: omitted or `with: all` lands every service together,
`with: none` gives a service a node of its own, and `with: <service>` puts it
on the same node as that service (`with: sim` below pins `recorder` beside
`sim`; naming a service that set `none` joins it away from the default group).
Services that land together form one placement group with one GPU class (type
only), and every service in a GPU group sees that GPU. Placement resolves over
the services a session runs (a profile-gated service places nothing until its
profile is active). This release runs one placement group per session and
refuses a session that lands services on different nodes, naming both services.

`antioch project topology` prints the resolved groups, node
class, and requests for that selection. An omitted `resources.cpu` or
`resources.memory` is no request and no limit: the service can use all the
compute of its machine. A declared value is what the service gets, placed with
that much and capped at it, so a real robot's compute budget can be simulated.
`stop_grace_period` (a duration such as `30s`) and `shm_size` (a memory
quantity such as `2Gi`) tune one service's shutdown and shared memory.
`tmpfs` (absolute paths) and `user` (`uid` or `uid:gid`) work on every cell.
`ipc: host`, `cap_add`, and `devices` are admitted only from a short list and
run only on a machine cell; a Kubernetes cell refuses them by name at
preparation, because its session namespace enforces the baseline Pod Security
standard.

```yaml
services:
  sim:
    build:
      context: .
      dockerfile: Dockerfile
    resources:
      gpu: rtx-pro-6000

  recorder:
    image: registry.example.com/robot/recorder:release
    with: sim
    resources:
      cpu: 2
      memory: 4Gi
```

CPU accepts whole units, decimal units, or milli-units. Memory accepts byte
quantities such as `512Mi`, `8Gi`, or `2G`.

GPU classes are a closed vocabulary. The current engine catalog supports only
`rtx-pro-6000`; `h100` and arbitrary class names are rejected during manifest
validation before submission.

Restart defaults to `never`. `on-failure` restarts a crashed service up to
three times. The retry count is fixed. `always` restarts a long-running
service after any exit.

## Declare named routes

Every port is a mapping with a name and integer port:

```yaml
services:
  sim:
    build: .
    resources:
      gpu: rtx-pro-6000
    ports:
      - name: viewer
        port: 8080
        protocol: tcp
        direction: client-to-service
      - name: callbacks
        port: 9000
        direction: service-to-client
```

TCP is the default protocol. UDP is also supported.

- `client-to-service` means the connected client starts the connection to the
  service.
- `service-to-client` means the service starts the connection to a listener
  registered by that client.

Use `antioch services ports --help` to bind or clear routes. Route names and
service endpoints must be unique, and platform routes are reserved.

## Use watch for interactive development

Watch rules react to project-relative local paths. They are a development
loop for a live interactive session. They are not part of scenario or suite
identity.

### Sync files

```yaml
watch:
  - action: sync
    path: src
    target: /workspace/project/src
    include: ["**/*.py"]
    ignore: ["**/__pycache__/**"]
```

`sync` copies the matching files to a target under `/workspace/project`.

### Sync, then restart

```yaml
watch:
  - action: sync+restart
    path: config
    target: /workspace/project/config
```

`sync+restart` copies the files, then restarts that service.

### Sync, then run a bounded command

```yaml
watch:
  - action: sync+exec
    path: schemas
    target: /workspace/project/schemas
    command: ["python", "-m", "tools.compile_schemas"]
    timeout: 30s
```

`sync+exec` requires a non-empty command and a timeout greater than zero and
no more than five minutes.

### Rebuild the service image

```yaml
watch:
  - action: rebuild
    path: Dockerfile
```

`rebuild` uses a project-relative path as its change trigger. When the path
changes, Antioch captures the complete declared build context, builds the new
image, and updates that service in the same session. It has no container
target.

Sync targets in one service cannot overlap. Each watch rule can use `include`
and `ignore` patterns.

Use `antioch services restart` when a change needs a manual service
restart. A `build` service can use a rebuild rule for dependency changes. A
published `image` service needs a new image reference and a new session.

## Pin content-addressed images

```yaml
services:
  autonomy:
    image: registry.example.com/robot/autonomy@sha256:<digest>
```

Publish the image through your registry before submission. Antioch mirrors a
registry image to an Antioch-owned digest before a run uses it.

Resolved image digests and run inputs are the repeatable identity. Project
source lives at `/workspace/project` for both `image:` and `build:`
services. Watch transfer keeps a live interactive session current. A
recorded scenario or suite run places the submitted project files at that same
path.

## Define suites

```yaml
suites:
  acceptance:
    description: Warehouse acceptance checks
    select:
      - tags: ["warehouse", "smoke"]
        exclude_tags: ["slow"]
      - scenarios: ["dock_alignment"]
        cases: ["narrow", "wide"]
```

Each selector can use paths, scenario names, case IDs, required tags, and
excluded tags. Fields inside one selector all apply. Separate selectors form
an ordered union.

Use `antioch suite collect` to verify the expansion before submission.
