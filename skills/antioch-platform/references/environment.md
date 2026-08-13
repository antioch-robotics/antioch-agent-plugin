# Projects and environments: `antioch.yaml`

## Start in a project

The project manifest is `antioch.yaml`. From the directory that contains it,
check the packaged CLI before setup or dispatch:

```bash
uv run antioch --version
uv run antioch init warehouse-amr
```

`init` is local. It reads the declared engine extra and installed SDK version, then
creates a manifest with the matching image coordinate, a simulation script,
example scenarios, `.gitignore`, and `.dockerignore`. It does not allocate a
machine or replace existing source files. Use `antioch init --help` for the
JSON result and directory behavior. The generated ignore file covers local
`home/`, `.cache/`, and `outputs/` trees so they do not enter a build context.
The coordinate pins the cloud engine and SDK version; edit it to upgrade an
existing project.

## Base image inventory

The published engine image is the base for a project Dockerfile. The
`isaac-601-ga` core starts from Ubuntu 24.04 and includes the Isaac Sim 6.0.1
runtime, Python 3 with pip, venv, and development headers, `uv`, `git`, and
`git-lfs`. It also includes the runtime graphics and audio libraries that Kit
needs: Vulkan and `vulkan-tools`, GL/EGL/GLES/GLVND, X11, and audio support.
The Isaac Sim layer carries the in-process ROS 2 Jazzy Python stack. The
`isaac-lab-30b2` layer adds Isaac Lab 3.0.0b2 and its pinned framework extras.

This is a runtime inventory, not a promise of a general build workstation.
The base does not include `gcc`/`g++` or `build-essential`, the FFmpeg command
line tool, graphics development headers, the full ROS 2 command-line and
message-build toolchain, or project-specific system packages. Install those
in the project Dockerfile when the project needs them. Verify an image that
you changed with `antioch services exec sim command -v TOOL` before dispatch.

The `FROM antioch-sim/<engine>:<sim-version>` line is the cache boundary. A
coordinate change invalidates the engine and every project layer. A changed
`apt-get` or `uv` instruction rebuilds that instruction and the layers after
it; a changed source `COPY` rebuilds only the later project layers. Keep
slow, stable system dependencies before source copies. Watch sync updates
source without rebuilding, while a queued or saved run freezes the final
image by digest.

The sim image coordinate is `antioch-sim/<engine>:<sim-version>`, stamped by
`antioch init`; the platform verifies the engine and SDK version from the image's
labels. Add a Dockerfile only for custom dependencies and use the same
coordinate in its `FROM` line. Install the public SDK from PyPI with the extra
that should select the initial engine image and examples:

```bash
uv add --compile-bytecode "antioch-sim[isaac-601-ga]"
```

The SDK wheel includes editor types for every supported engine. The extra does
not install Isaac locally or limit which types are available.

Do not add an engine selector to `antioch.yaml`; it has none.

## Keep the manifest small and complete

The manifest identifies the project, discovery paths, suites, and complete
service graph. The optional `schema_version` field defaults to the latest
schema. The required `services.sim` entry and auxiliary services share one
mapping:

```yaml
name: warehouse-amr
scenario_paths: [scenarios]

services:
  sim:
    image: antioch-sim/isaac-601-ga:0.3.31
    environment:
      ROS_DOMAIN_ID: "7"
    depends_on:
      ros: {condition: service_healthy}
    ports: ["8765:8765"]
  ros:
    image: ros:jazzy
    healthcheck:
      # Healthchecks skip the image entrypoint, so source the ROS
      # environment explicitly.
      test: ["CMD", "bash", "-c", "source /opt/ros/jazzy/setup.bash && ros2 topic list"]
      interval: 2s
      timeout: 5s
      retries: 15
```

Use the [Compose file reference](https://docs.docker.com/reference/compose-file/)
for the kept field vocabulary; Docker runs underneath Antioch. `services.sim`
is always active and receives platform-managed privileges, GPU access, init, and
the `/workspace/output` bind. Host network and IPC are defaults. An explicit
supported `network_mode` or `ipc` value can opt a service out when it needs
isolation. Auxiliary `profiles` select optional services; they are not allowed
on `sim`. Streaming is a CLI/runtime choice, not a manifest `stream` key.

Do not author `volumes`, `networks`, `restart`, `deploy`,
`scale`, `replicas`, `container_name`, `extends`, `include`, `secrets`,
`configs`, or `develop`. `watch` belongs directly under the service, not under
another development document. Environment names beginning with `ANTIOCH_` and
labels under `antioch.*` or `com.docker.*` are reserved.

## Watch versus an immutable run

Put development rules on the service:

```yaml
services:
  sim:
    build: .
    watch:
      - action: sync
        path: .
        target: /workspace/project
```

Start that loop with:

```bash
antioch services up --watch
```

The watcher arms before its initial reconciliation, batches file events,
propagates deletes, and reports failures instead of reconnecting silently. A
`sync+exec` or `sync+restart` rule affects only its service. `ports` open authenticated local tunnels
while the stack is up. Ctrl-C ends the watcher but leaves containers and declared ports
running; use `antioch services down` to stop them. A bare `antioch services up`
also opens declared ports and returns.

When no watcher is running, `antioch run`, `scenario run`, and
`suite run` perform one full current-tree reconciliation (including ignore and
`.dockerignore` rules) before dispatch. This is the run-freshness guarantee.

## Use private-registry images

An image service can use an image in your own registry:

```yaml
services:
  sim:
    image: ghcr.io/acme/warehouse-sim:2026.08
```

For an interactive pull, Antioch reads the same local Docker credential sources
as Docker (`auths`, `credsStore`, and `credHelpers`) and sends the credential
only with that Engine pull. For queued work, the submitter performs the
credentialed pull and pushes a digest-pinned copy into your organization's private registry
before the run is submitted:

```bash
antioch scenario run --scenario bin_pick --queue --json
```

Workers use the private image reference, so they do not need the third-party
registry credential. A missing local credential fails at the pull; Antioch
never asks a queued worker to guess or persist a team secret.

Scenario and suite run environments omit development `watch` rules and `ports`
tunnels. Queued runs save their exact digest-pinned environment before Antioch
distributes them. For a single-machine interactive run, Antioch captures the
admitted `sim` container's project workspace at dispatch and then attempts to
save the exact images and source the run used. When that capture and publish
succeeds, `antioch scenario rerun SCENARIO_RUN_ID` and
`antioch suite rerun SUITE_RUN_ID` queue the completed run again exactly as it
ran. Multi-machine interactive runs are not currently rerunnable. Repeat the
original command or use `--queue` when the result must be rerunnable. Antioch
explains when an older run or a failed source capture or publish leaves the
environment unavailable.

Write temporary frames, checkpoints, and debug files to `/workspace/output`.
Use `antioch services cp sim:/workspace/output/FILE ./FILE` for an explicit transfer and scenario artifacts or assets for
anything that must survive machine release.
