---
name: antioch-platform
description: >
  The entry point for any work with Antioch, the simulation platform for
  physical AI. Teaches projects, services, assigned
  GPU machines, scenarios, suites, assets, organizations, supported engines, and
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
  behavior can be tested, dispatch a scenario and inspect the saved results rather
  than asserting it.
- **Interactivity is needed** — poking at live simulator state, iterating
  cell by cell: start a Jupyter kernel (`references/sessions.md`) instead of
  round-tripping whole scripts.
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
the workflow and common examples; use `uv run antioch <command> --help` when a
command needs a detail not shown here.

Antioch keeps application code ordinary Isaac Python. The platform contributes
an assigned GPU machine, flexible project services, scenario history,
suite selection, and direct artifact access around that code.

## The project boundary

A project is rooted at `antioch.yaml`. Its optional `schema_version` field
defaults to the latest schema. The manifest owns the complete stack: one
required `services.sim` entry and the same `services` mapping for auxiliary
containers. `sim` is structural, not a label convention. The kept service keys
are deliberately familiar to [Docker's Compose file reference](https://docs.docker.com/reference/compose-file/),
and Docker runs underneath Antioch:
`build`, `image`, `command`, `entrypoint`, `environment`, `working_dir`,
`depends_on`, `healthcheck`, `profiles`, `labels`, `ports`, and `watch`.

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
      test: ["CMD", "ros2", "topic", "list"]
      interval: 2s
      timeout: 5s
      retries: 15
```

Antioch supplies all GPUs, init, restart policy `no`, and the sim output bind.
Host networking and IPC default to host mode; author an explicit supported
`network_mode` or `ipc` value such as `bridge`, `none`, or `private` when a
service needs an opt-out. Do not author `volumes`, `networks`, `restart`,
`deploy`, `scale`, `replicas`, `container_name`, `extends`, `include`,
`secrets`, `configs`, or `develop`; validation names the supported successor.
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

`services ps`, `services logs`, and `services down` are resolve-only. `services restart` and `services build` address the
existing project; check their help for the current service-selection behavior.

The direct connection verbs default to the required `services.sim` service:

```bash
uv run antioch services exec sim nvidia-smi
uv run antioch services ssh
uv run antioch services cp sim:/workspace/output/result.png ./result.png
```

Name an auxiliary service explicitly, for example
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

Multi-machine interactive runs are not currently rerunnable. Repeat the
original command or use `--queue` when the result must be rerunnable. Antioch
explains when an older run or a failed source capture or publish leaves the
environment unavailable.

## Deep references

Load the row whose trigger matches — proactively, the moment the trigger
appears, not after a failed guess:

| Reference | Reach for it when… |
|---|---|
| `references/auth.md` | signing in, a 401 or "not authenticated" error, checking who or which organization is active, or switching organizations — `antioch auth whoami --json` is the first move when anything auth-adjacent misbehaves |
| `references/authoring.md` | writing or editing an `@antioch.scenario` decorator — parameters, cases, `sim=` boot profiles — or local discovery refuses a scenario |
| `references/environment.md` | creating a project, editing `antioch.yaml`, adding or tuning a service, watch rules, tunnels, the sim image coordinate, or a manifest validation error |
| `references/running.md` | dispatching anything, choosing `antioch run FILE` vs a scenario, selection flags, queueing, streaming — and FIRST when a dispatched run misbehaves: pull its status and logs before theorizing |
| `references/scenarios.md` | debugging or validating behavior — list recent runs, inspect verdicts, checks, results, and artifacts; finding a run id, filtering history, or downloading an `.rrd` |
| `references/suites.md` | defining or running a suite, following or cancelling a queued suite, or reading last night's failures |
| `references/machines.md` | a runtime failure needs direct evidence — exec into a service, read container logs from the VM shell, copy files off — or an assignment, allocation, or release question |
| `references/json.md` | scripting any `--json` command, parsing list cursors, timestamps, repeatable filters, or a followed NDJSON stream |
| `references/ros2.md` | anything ROS 2 — `rclpy` in a scenario, topics, an auxiliary autonomy container, or missing ROS tooling in the image |
| `references/sessions.md` | interactivity is needed — iterate on live Isaac state cell by cell with a Jupyter kernel, stream a kernel, or point local JupyterLab at a remote kernel |
| `references/assets.md` | ANY asset need — the task needs a robot, prop, or environment; the user names an object that might exist as an asset; a result should be published for the team — search the asset catalog before building geometry by hand |

The one coding rule that crosses every Antioch project is import safety:
scenario modules must import cleanly with no simulator installed because the
CLI discovers them locally before requesting a machine.
