# Projects and environments

This file owns the project workflow: creating a project, choosing the
simulation image and SDK engine option, understanding the base image,
customizing it with a Dockerfile, using private registries, and saving images
for queued work. The complete
`antioch.yaml` schema — every key, constraint, watch rule, and example
manifest — is `manifest.md`; go there for any manifest authoring or
validation question.

## Start in a project

The project manifest is `antioch.yaml`. From the directory that contains it,
check the packaged CLI before setup or dispatch:

```bash
antioch --version
antioch init warehouse-amr
```

Run these commands with the project's Python environment active. In a uv
project whose environment is not active, prefix a command with `uv run`.

`init` is local and non-interactive. It reads the engine option declared in
the Python dependency and the installed SDK version, then creates a manifest
with the matching versioned simulation image, a simulation script,
example scenarios, `smoke` and `sweep` suites, `.gitignore`, and
`.dockerignore`. It does not allocate a
machine, register anything remotely, or replace existing source files, and it
refuses a directory that is already inside a project. Pass
`--engine isaac-sim-6.0.1` or `--engine isaac-lab-3.0` to override the engine
the installed extra selects. The generated ignore file covers local
`home/`, `.cache/`, and `outputs/` trees so they do not enter a build context.

Inspect project identity afterwards — useful when several checkouts exist or
a command acts on the wrong project:

```bash
antioch project current --json
antioch project list --json
```

`project current` prints the project the working directory selects (literal
`null` in JSON outside any project); `project show` names one by name or id.

A project needs at least one service. The optional name `services.sim`
identifies the simulation service. A service-only stack (viewer, API, ROS tooling)
is valid — Isaac commands refuse it until a `sim` service is declared. See
`manifest.md` for the full contract.

## Base image inventory

The published engine image is the base for a project Dockerfile. The
`isaac-sim-6.0.1` core starts from Ubuntu 24.04 and includes the Isaac Sim 6.0.1
runtime, Python 3 with pip, venv, and development headers, `uv`, `git`, and
`git-lfs`. It also includes the runtime graphics and audio libraries that Kit
needs: Vulkan and `vulkan-tools`, GL/EGL/GLES/GLVND, X11, and audio support.
The Isaac Sim layer carries the in-process ROS 2 Jazzy Python stack. The
`isaac-lab-3.0` layer adds Isaac Lab 3.0.0b2.post1 and its pinned framework
extras.

This is a runtime inventory, not a promise of a general build workstation.
The base does not include `gcc`/`g++` or `build-essential`, the FFmpeg command
line tool, graphics development headers, the full ROS 2 command-line and
message-build toolchain, or project-specific system packages. Install those
in the project Dockerfile when the project needs them. Verify an image that
you changed with `antioch services exec sim command -v TOOL` before dispatch.

The `FROM antioch-engine/<engine>:<sdk-version>` line is the first Docker image
layer. Changing it rebuilds the engine and every project layer. A changed
`apt-get` or `uv` instruction rebuilds that instruction and the layers after
it; a changed source `COPY` rebuilds only the later project layers. Keep
slow, stable system dependencies before source copies. Watch sync updates
source without rebuilding, while a queued or saved run freezes the final
image by digest.

## Simulation image and SDK version

`antioch init` writes `antioch-engine/<engine>` under `services.sim.image`.
The engine name selects Isaac Sim or Isaac Lab. Without a version tag, cloud
runs use the Antioch SDK release installed with the CLI, so local code and
the cloud simulation always match; add `:<sdk-version>` only to hold one
exact release. Add a Dockerfile only for custom dependencies and use the same
image with an explicit tag in its `FROM` line. Install the public SDK from
PyPI with the engine option that should select the first image and examples:

```bash
uv add --compile-bytecode "antioch-sim[isaac-sim]"
```

The SDK wheel includes editor types for every supported engine. The engine option does
not install Isaac locally or limit which types are available.

Do not add an engine selector to `antioch.yaml`; it has none. The built sim
image identifies the engine that runs in the cloud.

## Development flow versus an immutable run

Put watch rules on the service (`manifest.md` owns the rule schema and the
sync/exec/restart/rebuild decision table) and start the loop with:

```bash
antioch services up --watch
```

The watcher starts before its initial sync, batches file events,
propagates deletes, and reports failures instead of reconnecting silently.
`ports` open authenticated local tunnels
while the stack is up. Ctrl-C ends the watcher but leaves containers and
declared ports
running; use `antioch services down` to stop them. A bare `antioch services up`
also opens declared ports and returns.

When no watcher is running, `antioch run`, `scenario run`, and
`suite run` sync and verify the latest project files, including `.gitignore`
and `.dockerignore` rules, before they start.

## Use private-registry images

An auxiliary service can use an image in your own registry. The `sim`
service cannot: its `image` must stay an `antioch-engine/<engine>` image, or
its Dockerfile must start `FROM` one with an explicit `:<sdk-version>` tag —
see `manifest.md`.

```yaml
services:
  viz:
    image: ghcr.io/acme/warehouse-viz:2026.08
```

For an interactive pull, Antioch reads the same local Docker credential sources
as Docker (`auths`, `credsStore`, and `credHelpers`) and sends the credential
only with that Engine pull. For queued work, the submitter performs the
credentialed pull and saves the exact image digest in your organization's
private registry
before the run is submitted:

```bash
antioch scenario run --scenario bin_pick --queue --json
```

Workers use the private image reference, so they do not need the third-party
registry credential. A missing local credential fails at the pull; Antioch
never asks a queued worker to guess or persist a team secret.

Scenario and suite run environments omit development `watch` rules and `ports`
connections. Queued runs save their exact service images, project files, and
inputs before they start. Antioch also attempts to save those files and images
for a single-machine interactive run. When they are available,
`antioch scenario rerun SCENARIO_RUN_ID` and
`antioch suite rerun SUITE_RUN_ID` queue the completed run again exactly as it
ran. Multi-machine interactive runs are not currently rerunnable. Repeat the
original command or use `--queue` when the result must be rerunnable. Antioch
explains which files or images are unavailable when a run cannot be rerun.

Write temporary frames, checkpoints, and debug files to `/workspace/output`.
Use `antioch services cp sim:/workspace/output/FILE ./FILE` for an explicit
transfer and scenario artifacts or assets for
anything that must survive machine release. To keep a built service image
itself, export it to the local Docker daemon while the assignment is live:
`antioch services images pull sim` (`machines.md` covers the workflow).
