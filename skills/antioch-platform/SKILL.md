---
name: antioch-platform
description: >
  The entry point for any work with Antioch, the simulation platform for
  physical AI. Teaches projects, services, assigned
  GPU machines, scenarios, suites, assets, organizations, supported engines,
  Mission Control workspaces, and
  the CLI workflows that connect them — and routes the sibling Antioch skills.
  Load this first whenever a repository contains antioch.yaml, when the user
  mentions Antioch, before running an antioch command, and whenever a task needs
  a platform capability: finding or publishing an asset (a robot, prop, or
  environment the user names), diagnosing a failed or misbehaving run, reading
  run history, logs, and artifacts, direct machine access, or an interactive
  Jupyter session. Not for simulation-substrate API detail (antioch-research),
  scenario verdict and telemetry design (scenario-design), or Isaac code itself
  (isaac-sim-6, isaac-lab-3) — load it first even then, for dispatch and
  read-back.
---

# The Antioch platform

## The first move is the CLI

The workspace or checkout you are in is a seat for dispatching work to GPU
machines. It is not an unknown Linux box. For any Antioch task, your first
substantive action is a formed `antioch` command, not filesystem or package
exploration. The platform reports its own state; ask it.

The default loop for any task:

```bash
uv run antioch scenario collect --json      # 1. what is authored here?
uv run antioch machine status               # 2. what is my seat attached to?
uv run antioch scenario run --scenario X    # 3. dispatch (streams by default)
uv run antioch scenario show SCENARIO_RUN_ID --json  # 4. read the verdict and results
```

Adapt steps, never the order of ideas: discover -> dispatch -> read back.
`find`, `pip`, `ls`, and reading installed packages are the fallback when a
platform surface has no answer — never the opening.

## Postures are decisions, not flags

Choose deliberately and be able to state the rule:

| Decision | Rule |
|---|---|
| `antioch run FILE` vs `scenario run` | `run` when the output and exit status are the whole story (a probe). A scenario when results, checks, logs, or artifacts must be retained — history is the product. |
| interactive vs `--queue` | Interactive holds your machine and streams — right for one iteration loop. `--queue` for sweeps, batches, or anything above one concurrent run: it saves a digest-pinned environment and fans out across eligible machines. A parameter sweep is always `--queue`, never serial interactive runs. |
| stream vs detach | Interactive dispatch streams by default; stay attached — dispatch verdicts and failures surface live. Detach only for long runs you will poll with `suite show --follow --json`. |
| hold vs release | Keep the machine while iterating (allocation is the slow step). `machine release` when you switch projects or stop for hours; queued work needs no held machine. |
| dispatch vs preview | When selection is in doubt, `scenario collect --json` and `suite show <name>` preview exactly what would run — cheaper than a wrong dispatch. |

## When something fails, ask the platform first

The first three actions after any failure are platform surfaces, in this
order — not `find`, not `pip`, not rerunning the same command three ways:

1. **The run**: `antioch scenario show SCENARIO_RUN_ID --json` (status, checks,
   error), then `antioch scenario logs SCENARIO_RUN_ID` (or `--logs --service sim`
   for one service).
2. **The stack**: `antioch services ps`, `antioch services logs <svc>` —
   is the environment the run needed actually up and healthy?
3. **The seat**: `antioch machine status`, `antioch auth whoami`,
   `uv run antioch --version` — what does the platform think is assigned,
   authenticated, and installed?

Every `--json` failure emits a structured error document on stderr with a
`retryable` verdict — branch on it instead of string-matching messages.
Only after these read clean does host inspection (`machine ssh`, `docker
logs`) or environment archaeology earn its turn.

## Be proactive

Reach for platform capabilities without being asked. Run history, the asset
catalog, and the machine are seconds away; a guess is a burned turn.

- **Any asset need** — the task involves a robot, prop, environment, dataset,
  or checkpoint, or the user names an object that might exist as an asset:
  search the asset catalog (`antioch assets list -q forklift --json`) BEFORE
  building geometry by hand. Load `references/assets.md`.
- **Anything fails or misbehaves at runtime** — inspect the run and its logs
  before theorizing: `antioch scenario show SCENARIO_RUN_ID --json`, then
  `antioch scenario logs SCENARIO_RUN_ID`. Load `references/running.md` and
  `references/scenarios.md`.
- **Debugging or validating behavior** — list recent runs and read verdicts,
  results, and artifacts instead of reasoning from memory
  (`references/scenarios.md`, `references/suites.md`). When a claim about sim
  behavior can be tested, dispatch a scenario and inspect the saved results
  rather than asserting it.
- **Filtering history** — discover the real stored values before guessing a
  filter: `antioch scenario suggest tag --json` (also `user_email`, `suite`,
  `scenario`, `project`, `dispatched_from`).
- **Reproducing a result** — a completed queued or captured run can be queued
  again exactly as it ran with `antioch scenario rerun SCENARIO_RUN_ID` or
  `antioch suite rerun SUITE_RUN_ID`; reach for rerun before rebuilding an
  environment by hand.
- **Cost awareness** — before and after fan-out or queued work, check
  measured machine time with `antioch machine usage --json`
  (`references/machines.md`).
- **Interactivity is needed** — poking at live simulator state, iterating
  cell by cell: start a Jupyter kernel (`references/sessions.md`) instead of
  round-tripping whole scripts.
- **Running inside Mission Control** — `ANTIOCH_WORKSPACE_ID` in the
  environment means the hosted workspace seat: ephemeral filesystem, refused
  auth verbs, `dispatched_from` provenance. Load
  `references/mission-control.md` the moment that variable appears.
- **Any question about the simulation substrate** — Isaac Sim, Isaac Lab,
  Omniverse, USD, PhysX, Rerun APIs and behavior: call the `antioch-research`
  MCP tools liberally instead of guessing or reading platform docs.

## Which skill, when

This skill is the always-first entry point; route the rest by trigger:

| Trigger | Load |
|---|---|
| Platform and CLI work — projects, machines, dispatch, run history, suites, assets, sessions. "Find me a warehouse asset", "why did my run fail" | this skill + the reference table below |
| Designing or reviewing a scenario or suite — cases, `run.check` verdicts, telemetry, blueprints, `.rrd` evidence. "Design a pick-and-place scenario" | `scenario-design` |
| Writing, porting, or debugging Isaac Sim code — scenes, physics, sensors, navigation, manipulation, rendering, SDG. "How do I make a camera" | `isaac-sim-6`, grounded by `antioch-research` |
| Isaac Lab 3 environments, managers, and RL training; porting 2.x Lab code | `isaac-lab-3`, grounded by `antioch-research` |
| Any substrate API or behavior question — Isaac Sim/Lab, Omniverse/Kit, OpenUSD, PhysX, Warp, Rerun, and the rest of the index. "What does this USD attribute mean" | `antioch-research` MCP tools |

## Start with the project CLI

Run commands from the project directory and keep the project interpreter
explicit while checking the installation:

```bash
uv run antioch --version
uv run antioch --help
```

The help output is the authority for flags and subcommands. This skill teaches
the workflow and common examples; `references/cli.md` maps the complete
command tree, the JSON and exit-code contracts, and the plays agents
underuse, and `uv run antioch <command> --help` settles any remaining detail.

Antioch keeps application code ordinary Isaac Python. The platform contributes
an assigned GPU machine, flexible project services, scenario history,
suite selection, and direct artifact access around that code.

## The project boundary

A project is rooted at `antioch.yaml`. The manifest owns the complete stack:
one `services` mapping that must declare at least one service. `sim` is the
reserved simulation service — structural, not a label convention. It is
**optional**: a service-only stack (a viewer, an API, ROS tooling) is valid,
but the commands that run Isaac code — `antioch run`, scenarios, suites,
Jupyter — refuse a stack without it. The kept service keys are deliberately
familiar from [Docker's Compose file reference](https://docs.docker.com/reference/compose-file/),
and Docker runs underneath Antioch:
`build`, `image`, `command`, `entrypoint`, `environment`, `working_dir`,
`depends_on`, `healthcheck`, `profiles`, `labels`, `ports`, `watch`,
`network_mode`, `ipc`, `cpus`, `mem_limit`, and `privileged`.
`references/manifest.md` is the complete schema — every field, default,
constraint, rejected key with its remedy, and worked example manifests.

```yaml
name: warehouse-amr

services:
  sim:
    build: .
    environment: {ROS_DOMAIN_ID: "7"}
    ports: ["8765:8765"]
    watch:
      - action: sync
        path: .
        target: /workspace/project
        ignore: [".git/", ".venv/", "__pycache__/"]
  ros:
    image: ros:jazzy
    healthcheck:
      test: ["CMD", "bash", "-c", "source /opt/ros/jazzy/setup.bash && ros2 topic list"]
      interval: 2s
      timeout: 5s
      retries: 15
```

Antioch supplies all GPUs, init, restart policy `no`, and the output bind.
Networking and IPC default to host mode; opt a service out with an explicit
supported value — `network_mode` accepts `host`, `none`, or `bridge`, `ipc`
accepts `host`, `none`, `private`, or `shareable`, and both accept
`service:NAME` references. Do not author `volumes`, `networks`, `restart`,
`deploy`, `scale`, `replicas`, `gpus`, `init`, `container_name`, `extends`,
`include`, `secrets`, `configs`, or `develop`; validation names the supported
successor for each (`references/manifest.md` tabulates them).
The manifest has no `stream` key: choose streaming at runtime. `ports` are
authenticated local tunnels to host-network ports, not a service-name network.
Environment names beginning with
`ANTIOCH_` and labels in `antioch.*` or `com.docker.*` are reserved.

The sim image coordinate `antioch-sim/<engine>:<sim-version>` selects the
engine (`isaac-601-ga` or `isaac-lab-30b2`) and pins its Antioch SDK version;
edit the coordinate to upgrade. The engine extra used to install the SDK tells
`antioch init` which coordinate and examples to write first. The public SDK
wheel includes typing for every supported engine. Add a Dockerfile only for
custom dependencies and use the selected coordinate in its `FROM` line. The
platform verifies the engine and SDK version from the image's labels.
Keep `pxr`, `omni`, `carb`, `isaacsim`, and `isaaclab*`
imports lazy so local scenario discovery works without a simulator installed.

## Objects and ownership

- **Project** — the directory containing `antioch.yaml` and its Dockerfiles.
- **Machine** — an ephemeral GPU VM assigned to one user and project. Its
  scratch filesystem is not a durable source store.
- **Scenario** — a decorated Python function selected locally and executed as
  a recorded simulation run.
- **Suite** — an ordered union of selector clauses saved in `antioch.yaml`.
- **Asset** — an immutable file revision shared with the organization.
- **Organization** — the visibility boundary for scenario history, suite
  history, assets, and usage. Machine assignments remain personal.
- **Mission Control workspace** — the hosted authoring seat in the webapp:
  one ephemeral per-user environment with the CLI signed in, examples, hosted
  JupyterLab, and an agent terminal (`references/mission-control.md`).

## The useful loops

Start a development stack with a foreground watcher:

```bash
uv run antioch services up --watch
```

`services up` may allocate or reuse the project's machine, builds changed services,
starts dependencies, and waits for their health checks. `--watch` arms file rules
and stays in the foreground. Declared ports remain reachable while the stack is up.
Ctrl-C ends that watch session but leaves containers and declared ports running; use
`antioch services down` to stop them. A bare `services up` also opens declared ports.
`build`, `run`, scenario/suite dispatch, and Jupyter start are the other
operations that may allocate or prepare a machine. Without a running watcher,
each run reconciles the current tree once before dispatch, so it never silently
uses stale source.

For authoring or reviewing a scenario — declaring cases, modelling pass/fail
with `run.check`, logging scalars/images/transforms, the live-vs-recorded
`.rrd` flow, and viewer layouts — load the `scenario-design` skill.

Observe or tear down an existing stack without allocation:

```bash
uv run antioch services ps
uv run antioch services logs sim
uv run antioch services down
uv run antioch machine status
```

`services ps`, `services logs`, and `services down` are resolve-only.
`services restart` and `services build` address the
existing project; check their help for the current service-selection behavior.

Direct container access names the service; `ssh` alone defaults to `sim` when
that service exists:

```bash
uv run antioch services exec sim nvidia-smi
uv run antioch services ssh
uv run antioch services cp sim:/workspace/output/result.png ./result.png
```

`exec` and `cp` always take an explicit service, for example
`uv run antioch services exec ros bash -c "source /opt/ros/jazzy/setup.bash && ros2 topic list"`
(`exec` skips the image entrypoint, so source the environment a stock ROS image
prepares there). These verbs resolve an existing
assignment and do not allocate one. `uv run antioch machine ssh` targets the VM shell.

Preview authored definitions before dispatching:

```bash
uv run antioch scenario collect --json
uv run antioch scenario run --scenario pick_and_place
uv run antioch scenario run --tag warehouse --path scenarios
```

Inspect a completed run and its saved outputs:

```bash
uv run antioch scenario show SCENARIO_RUN_ID --json
uv run antioch scenario logs SCENARIO_RUN_ID
uv run antioch scenario show SCENARIO_RUN_ID --logs --service sim
uv run antioch scenario download SCENARIO_RUN_ID
```

The webapp has the same per-service log selector. `antioch machine status`
surfaces the direct machine URL and stream address when available. For host
diagnostics, run `antioch machine ssh`; Docker runs underneath the stack, so raw
`docker ps`, `docker logs`, and `docker exec` are supported from that VM shell.

Use `antioch run FILE` for a one-off probe whose output and exit status belong
to the process. Use a decorated scenario when results, telemetry, logs, or
artifacts must be retained.

Suites are named selections, not positional scenario selectors:

```bash
uv run antioch suite run acceptance
uv run antioch suite run acceptance --queue --json
uv run antioch suite show SUITE_RUN_ID --follow --json
```

Queueing saves a digest-pinned environment before Antioch distributes the
scenario runs across eligible machines. Development `watch` rules and tunnels
do not enter that immutable environment.

## Reruns

Queued runs save their exact digest-pinned environment before Antioch
distributes them. For a single-machine interactive run, Antioch captures the
admitted `sim` container's project workspace at dispatch and then attempts to
save the exact images and source the run used. When that capture and publish
succeeds, queue the completed run again exactly as it ran — same images,
parameters, and cases, with a fresh identity:

```bash
uv run antioch scenario rerun SCENARIO_RUN_ID
uv run antioch suite rerun SUITE_RUN_ID
```

The webapp's Re-run button is the GUI twin of these commands.
Multi-machine interactive runs are not currently rerunnable. Repeat the
original command or use `--queue` when the result must be rerunnable. Antioch
explains when an older run or a failed source capture or publish leaves the
environment unavailable.

## Deep references

Load the row whose trigger matches — proactively, the moment the trigger
appears, not after a failed guess:

| Reference | Reach for it when… |
|---|---|
| `references/manifest.md` | authoring or editing ANY part of `antioch.yaml` — the complete schema: every key, default, and constraint, the sim service contract, build and watch mechanics, rejected keys with remedies, and worked example manifests |
| `references/cli.md` | orienting on the CLI as a whole — the full command map with options and JSON shapes, global environment variables, the credential store, stdout/stderr and exit-code contracts, TTY traps, and the proactive plays agents underuse |
| `references/environment.md` | creating a project, choosing the engine coordinate or SDK extra, the base image inventory and Dockerfile layering, private registries, or the queue image-copy story — it points at `manifest.md` for schema detail |
| `references/auth.md` | signing in, a 401 or "not authenticated" error, checking who or which organization is active, switching organizations, or a wrong-account/wrong-API symptom — `ANTIOCH_ENV` and the per-environment credential layout live here |
| `references/authoring.md` | writing or editing an `@antioch.scenario` decorator — parameters, cases, `sim=` boot profiles — or local discovery refuses a scenario; verdict and telemetry design belongs to the `scenario-design` skill |
| `references/running.md` | dispatching anything, choosing `antioch run FILE` vs a scenario, selection flags, queueing and its flag constraints, streaming, output verbosity — and FIRST when a dispatched run misbehaves: pull its status and logs before theorizing |
| `references/scenarios.md` | debugging or validating behavior — list recent runs, inspect verdicts, checks, results, and artifacts; finding a run id, filtering history with suggest and predicates, cancelling, or downloading an `.rrd` |
| `references/suites.md` | defining or running a suite, following or cancelling a queued suite, comparing suite runs, or reading last night's failures |
| `references/machines.md` | a runtime failure needs direct evidence — exec into a service, read container logs, copy files off — or an assignment, allocation, release, usage-reporting, or image-export question |
| `references/ros2.md` | anything ROS 2 — `rclpy` in a scenario, topics, an auxiliary autonomy container, or missing ROS tooling in the image |
| `references/sessions.md` | interactivity is needed — iterate on live Isaac state cell by cell with a Jupyter kernel, stream a kernel, or point local JupyterLab at a remote kernel |
| `references/assets.md` | ANY asset need — the task needs a robot, prop, or environment; the user names an object that might exist as an asset; a result should be published for the team — search the asset catalog before building geometry by hand |
| `references/mission-control.md` | the hosted workspace seat — `ANTIOCH_WORKSPACE_ID` is set, a run's origin says `Mission Control`, auth verbs are refused, or the user works from the webapp console |

The one coding rule that crosses every Antioch project is import safety:
scenario modules must import cleanly with no simulator installed because the
CLI discovers them locally before requesting a machine.
