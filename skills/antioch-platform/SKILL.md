---
name: antioch-platform
version: "1.1.12"
description: >
  The entry point for any work with Antioch, the simulation platform for
  physical AI. Teaches projects, services, sessions, scenarios,
  suites, assets, organizations, supported engines, Mission Control,
  and the CLI workflows that connect them — and routes the sibling
  Antioch skills. Load this first whenever a repository contains antioch.yaml,
  when the user mentions Antioch, before running an antioch command, and
  whenever a task needs a platform capability: finding or publishing an asset
  (a robot, prop, or environment the user names), diagnosing a failed or
  misbehaving run, reading run history, logs, and artifacts, direct session
  access, or an interactive Jupyter session. Not for simulation-substrate API
  detail (antioch-research), scenario verdict and telemetry design
  (scenario-design), or Isaac code itself (isaac-sim-6, isaac-lab-3) — load it
  first even then, for running work and reading the results.
---

# The Antioch platform

## Start with the current project

Run the CLI from the directory that contains `antioch.yaml`. Activate the
project's Python environment first, or prefix commands with `uv run`.

```bash
antioch --version
antioch --help
```

The help output is the authority for subcommands and options. This skill
teaches workflows. Check help before using an unfamiliar command:

```bash
antioch session --help
antioch services --help
antioch services exec --help
antioch jupyter --help
antioch scenario run --help
antioch suite run --help
```

## Choose the correct workflow

| Need | Workflow |
|---|---|
| One Python process and its exit status | `antioch services exec python src/main.py` |
| Fast work against live services | Start or reuse an interactive session |
| Cell-by-cell Isaac work | Use JupyterLab with a kernel in the simulator service |
| A saved simulation test | Submit a scenario |
| A saved group of simulation tests | Submit a suite |

All simulation compute runs in sessions. A session contains one project's
services and is either interactive or background:

- An interactive session belongs to one user. `services exec`, direct service
  access, watch actions, routes, attached scenarios and suites, and Jupyter
  use it.
- A detached scenario or suite uses reusable background sessions. Scenario
  and suite records keep results and progress independently; they do not own
  compute or usage.

Mission Control authoring uses a separate ephemeral workspace. A workspace is
not a session and never runs the simulator.

An interactive run uses one selected live session. The CLI prefers an
explicit `--session SESSION`, then the session it last used in this worktree,
then the sole live interactive session. It stops and asks you to choose when
several compatible sessions are available. For detached work, Antioch reuses
compatible background sessions pinned to the submitted project revision or
starts more when needed.

Scenario and suite commands use the selected interactive session and stay
attached until the run finishes. Add `--detach` for unattended work that must
continue after the terminal closes. Add `--follow` with `--detach` to watch
the detached submission until it completes.

## Understand the project

A project is rooted at `antioch.yaml`. The file declares one or more services.
The simulator is a role derived from verified engine-image metadata, not a
service name. Scenarios, suites, and Jupyter require that role. A project
without it can still run ordinary commands and Python files. With several
engine services, select one with `x-antioch: {runner: true}`. `antioch init`
uses `sim` as an example name; any valid service name works.

```yaml
id: warehouse-amr-0123456789abcdef
name: warehouse-amr

services:
  sim:
    build:
      context: .
      dockerfile: Dockerfile
    resources:
      gpu: rtx-pro-6000
    ports:
      - name: viewer
        port: 8080
        direction: client-to-service
    watch:
      - action: sync
        path: .
        target: /workspace/project
      - action: rebuild
        path: Dockerfile

  autonomy:
    image: registry.example.com/robot/autonomy:release
    depends_on:
      sim:
        condition: service_started
```

Service names are also network names, so one service can reach another on a
declared `client-to-service` port. ROS 2 multicast and shared memory cannot
cross the isolated service namespaces. A Fast DDS discovery server can use
`ROS_DISCOVERY_SERVER=service:port`; `ROS_STATIC_PEERS` instead names peer hosts,
not discovery-server ports. Neither setting alone proves DDS data delivery.
Use [the ROS 2 reference](references/ros2.md) for transport requirements and
the simpler in-process or explicit bridge options.

A session runs on one machine: one service declares the
GPU class in `resources.gpu`, every service in the session lands beside it
and sees that GPU, and an engine service with no class gets its
engine's recommended one.

Every named port declares who starts the connection.
`client-to-service` exposes a service endpoint to a connected client.
`service-to-client` lets a service reach a listener on that client. There is
no public service IP address to manage.

The development watch actions are `sync`, `sync+restart`, `sync+exec`, and
`rebuild`. Watch actions update a live interactive session. They do not affect
a scenario or suite run. Load
`references/manifest.md` before editing this file.

## Use image identity

A service declares exactly one published image or local build. Antioch
resolves every active service to an exact content digest and stores the whole
service graph as an immutable project revision. Sessions and runs pin that
revision. A local build is skipped only when its content-derived build key
resolves to an immutable registry digest with matching labels.

- An Antioch engine image reference selects the engine and SDK release.
- A registry image is mirrored to an Antioch-owned digest before admission.
- A local build captures the declared Docker context and produces a project
  image with an exact digest.

Project source lives at `/workspace/project` for `image:` and `build:`
services. A development watch rule transfers edits into a live session.
A recorded scenario or suite run places the submitted project files at that
same path.

Keep `pxr`, `omni`, `carb`, `isaacsim`, and `isaaclab*` imports inside
functions or under `if TYPE_CHECKING:`. Local discovery must work without
Isaac installed.

## Work with an interactive session

Start a session, or let `services exec` start one:

```bash
antioch session start
antioch session list
antioch session status
antioch services exec python src/main.py
antioch services exec --service autonomy -- ros2 topic list
```

`services exec` runs literal argv in the simulator service, or the only active
service when there is no simulator. Use `--service` to select a helper; the
command's first token never selects a service. Use repeated `--profile` options
to activate authored profiles when starting a session. Native commands stream
output and exit status without creating run history. Exec forwards stdin and
uses a terminal when local stdin and stdout are terminals; `--tty` and
`--no-tty` override that choice. Non-terminal stdout and stderr stay separate.
Commands that select a session accept `--session SESSION`;
without it the CLI uses the session it last used in this worktree, then the
sole live interactive session. If selection remains ambiguous, the CLI stops
and asks the user to choose.

Take direct evidence from existing services:

```bash
antioch services exec --session SESSION --service sim -- nvidia-smi
antioch services logs --session SESSION SERVICE...
antioch services cp sim:/workspace/project/output.png ./output.png
antioch shell --session SESSION
```

Use `antioch services watch` for a continuous development update loop. Use
`antioch services restart` to restart selected service processes.

Stop the session when the work is done:

```bash
antioch session stop --session SESSION
```

Stopping a session stops its services and removes its temporary files. A
session can also stop automatically after it has been idle.

## Use Jupyter

JupyterLab runs in the selected interactive session's simulator service. Kernel and cell
commands use that server's standard Jupyter APIs:

```bash
antioch jupyter lab
antioch jupyter cell '1 + 1'
antioch jupyter cell --stream 'import antioch; antioch.start_simulation()'
antioch jupyter lab --stop
```

Jupyter uses the selected interactive session and never starts one; add
`--session SESSION` to either command to name another session. `jupyter cell`
runs on the one live kernel and starts one when none is live; JupyterLab's own
controls manage several. Interrupt `antioch jupyter lab` or run
`antioch jupyter lab --stop` to stop JupyterLab. Load `references/sessions.md`
for the complete workflow. A kernel is headless; pass `--stream` on the cell
that calls `antioch.start_simulation()` when the user wants the Isaac GUI in
Mission Control.

## Submit and inspect evaluation runs

Preview scenario and suite selection on the client:

```bash
antioch scenario collect --json
antioch suite collect --json
```

Then submit work:

```bash
antioch scenario run --scenario pick_and_place
antioch suite run acceptance
```

A submitted run saves its selected inputs and the exact service images before
execution. Later local edits cannot change it.

Read the saved evidence before making a claim:

```bash
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario show SCENARIO_RUN_ID --logs
antioch scenario download SCENARIO_RUN_ID
antioch suite show SUITE_RUN_ID --json
```

A rerun gets a new ID and uses the saved service images and exact inputs. This
does not guarantee the same outcome or timing when scheduling, capacity,
simulator timing, or external assets differ:

```bash
antioch scenario rerun SCENARIO_RUN_ID
antioch suite rerun SUITE_RUN_ID
```

## Diagnose a failure from evidence

Use this order:

1. Read the run result and logs.
2. If the failure is interactive, read the selected session and service state.
3. Confirm the active identity and CLI version.

```bash
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario show SCENARIO_RUN_ID --logs
antioch session status --session SESSION --json
antioch auth whoami
antioch --version
```

Structured failures include a `retryable` verdict. Use it instead of matching
terminal text.

Search stored values before guessing a history filter:

```bash
antioch scenario suggest tag --json
```

## Find an asset before building content

When a task needs a robot, prop, environment, dataset, or checkpoint, search
the shelf first. Every word must appear in an asset's name or description:

```bash
antioch assets list -q "mobile robot" --json
antioch assets show robots/example --json
```

Read `name`, `description`, `scope`, and `latest_version` from the JSON, then
load the asset in the scenario by name with a pinned version. Load
`references/assets.md` for the whole find, load, measure, publish, and repair
workflow.

## Understand Mission Control

`ANTIOCH_WORKSPACE_ID` means the process runs in a Mission Control workspace.
Use the identity and project environment that Mission Control provides. Its
files and processes are temporary.

Mission Control can submit scenarios and suites. Those records retain their
evidence and use the same interactive or background session paths as a local
client. Load `references/mission-control.md` when the environment variable is
present.

## Route to the sibling skills

| Trigger | Load |
|---|---|
| Scenario checks, cases, telemetry, artifacts, viewer layout, or `.rrd` evidence | `scenario-design` |
| Isaac Sim scenes, physics, sensors, navigation, manipulation, rendering, or synthetic data | `isaac-sim-6`, then `antioch-research` |
| Isaac Lab environments, managers, tasks, or RL training | `isaac-lab-3`, then `antioch-research` |
| Any Isaac, Omniverse, OpenUSD, PhysX, Warp, or Rerun API detail | `antioch-research` |

## Deep guides

Load the guide that matches the task:

| Guide | Use it for |
|---|---|
| `references/manifest.md` | Project services, dependencies, resources, ports, images, and watch actions |
| `references/cli.md` | Command discovery, output contracts, and common command sequences |
| `references/environment.md` | SDK and engine selection, published images, and registry images |
| `references/auth.md` | Login, identity, organizations, and environment selection |
| `references/authoring.md` | Scenario decorators, parameters, cases, and local discovery |
| `references/running.md` | Choosing native, scenario, or suite execution and diagnosing dispatch |
| `references/scenarios.md` | Scenario history, evidence, cancellation, rerun, and download |
| `references/suites.md` | Suite selectors, execution, follow, cancellation, and comparison |
| `references/sessions.md` | Session lifecycle, direct access, routes, watch, and Jupyter |
| `references/ros2.md` | ROS 2 service and scenario workflows |
| `references/assets.md` | Asset search, pull, publish, verify, and repair |
| `references/mission-control.md` | Hosted authoring and dispatch from Mission Control |
