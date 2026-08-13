# Projects and environments

This file owns the project workflow: creating a project, the engine
coordinate and SDK extras, the base image inventory, Dockerfile layering,
private registries, and the queue image-copy story. The complete
`antioch.yaml` schema — every key, constraint, watch rule, and example
manifest — is `manifest.md`; go there for any manifest authoring or
validation question.

## Start in a project

The project manifest is `antioch.yaml`. From the directory that contains it,
check the packaged CLI before setup or dispatch:

```bash
uv run antioch --version
uv run antioch init warehouse-amr
```

`init` is local and non-interactive. It reads the declared engine extra and
installed SDK version, then
creates a manifest with the matching image coordinate, a simulation script,
example scenarios, `smoke` and `sweep` suites, `.gitignore`, and
`.dockerignore`. It does not allocate a
machine, register anything remotely, or replace existing source files, and it
refuses a directory that is already inside a project. Pass
`--engine isaac-601-ga` or `--engine isaac-lab-30b2` to override the engine
the installed extra selects. The generated ignore file covers local
`home/`, `.cache/`, and `outputs/` trees so they do not enter a build context.

Inspect project identity afterwards — useful when several checkouts exist or
a command acts on the wrong project:

```bash
uv run antioch project current --json
uv run antioch project list --json
```

`project current` prints the project the working directory selects (literal
`null` in JSON outside any project); `project show` names one by name or id.

A project needs at least one service; `services.sim` is the reserved,
optional simulation service. A service-only stack (viewer, API, ROS tooling)
is valid — Isaac commands refuse it until a `sim` service is declared. See
`manifest.md` for the full contract.

## Base image inventory

The published engine image is the base for a project Dockerfile. The
`isaac-601-ga` core starts from Ubuntu 24.04 and includes the Isaac Sim 6.0.1
runtime, Python 3 with pip, venv, and development headers, `uv`, `git`, and
`git-lfs`. It also includes the runtime graphics and audio libraries that Kit
needs: Vulkan and `vulkan-tools`, GL/EGL/GLES/GLVND, X11, and audio support.
The Isaac Sim layer carries the in-process ROS 2 Jazzy Python stack. The
`isaac-lab-30b2` layer adds Isaac Lab 3.0.0b2.post1 and its pinned framework
extras.

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

## Engine coordinate and SDK version

The sim image coordinate is `antioch-sim/<engine>:<sim-version>`, stamped by
`antioch init`; the platform verifies the engine and SDK version from the
image's labels. The coordinate pins the cloud engine and SDK version; edit it
to upgrade an existing project. Add a Dockerfile only for custom dependencies
and use the same coordinate in its `FROM` line. Install the public SDK from
PyPI with the extra that should select the initial engine image and examples:

```bash
uv add --compile-bytecode "antioch-sim[isaac-601-ga]"
```

The SDK wheel includes editor types for every supported engine. The extra does
not install Isaac locally or limit which types are available.

When the locally installed SDK version differs from the image coordinate's,
the CLI warns and the **image remains authoritative** — the code runs against
the image's SDK. Align the local pin with the coordinate to silence the
warning and keep editor types exact.

Do not add an engine selector to `antioch.yaml`; it has none. The built sim
image's labels are the engine source of truth.

## Development flow versus an immutable run

Put watch rules on the service (`manifest.md` owns the rule schema and the
sync/exec/restart/rebuild decision table) and start the loop with:

```bash
antioch services up --watch
```

The watcher arms before its initial reconciliation, batches file events,
propagates deletes, and reports failures instead of reconnecting silently.
`ports` open authenticated local tunnels
while the stack is up. Ctrl-C ends the watcher but leaves containers and
declared ports
running; use `antioch services down` to stop them. A bare `antioch services up`
also opens declared ports and returns.

When no watcher is running, `antioch run`, `scenario run`, and
`suite run` perform one full current-tree reconciliation (including
`.gitignore` and
`.dockerignore` rules) before dispatch. This is the run-freshness guarantee.

## Use private-registry images

An auxiliary service can use an image in your own registry. The `sim`
service cannot: its `image` must stay an `antioch-sim/<engine>:<semver>`
coordinate (or a Dockerfile build FROM one) — see `manifest.md`.

```yaml
services:
  viz:
    image: ghcr.io/acme/warehouse-viz:2026.08
```

For an interactive pull, Antioch reads the same local Docker credential sources
as Docker (`auths`, `credsStore`, and `credHelpers`) and sends the credential
only with that Engine pull. For queued work, the submitter performs the
credentialed pull and pushes a digest-pinned copy into your organization's
private registry
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
Use `antioch services cp sim:/workspace/output/FILE ./FILE` for an explicit
transfer and scenario artifacts or assets for
anything that must survive machine release. To keep a built service image
itself, export it to the local Docker daemon while the assignment is live:
`antioch services images pull sim` (`machines.md` covers the workflow).
