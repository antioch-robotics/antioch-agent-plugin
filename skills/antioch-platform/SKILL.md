---
name: antioch-platform
version: "1.2.19"
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
antioch service --help
antioch service exec --help
antioch jupyter --help
antioch scenario run --help
antioch suite run --help
```

For a requested local SDK/plugin install or update, use `antioch setup` when
the installed CLI exposes it. `antioch setup --dry-run` inspects without
installation or configuration changes. `antioch setup` targets a global uv tool,
configures every agent on PATH, and always checks sign-in and Research; a
failed check exits 1 with the next step. Explicit `antioch project update` in a
project updates active direct SDK dependencies, literal engine tags, native uv
lock/sync, and changed builds. Neither command starts or changes a session or
edits a shell profile. Do not turn an inspection request into an update. See
[CLI workflows](references/cli.md) for environment selection, the setup JSON
shape, and older-deployment limits. Read component states and action results,
not an aggregate `configured` flag or command trace.

## Choose the correct workflow

| Need | Workflow |
|---|---|
| One Python process and its exit status | `antioch service exec python src/main.py` |
| Fast work against live services | Start or reuse an interactive session |
| Cell-by-cell Isaac work | Use JupyterLab with a kernel in the simulator service |
| A saved simulation test | Submit a scenario |
| A saved group of simulation tests | Submit a suite |
| Record Python work on your own compute | Call a source-backed scenario or enter `Scenario` |
| Which CLI, deployment, plugin, project SDK, engine, or session version is in play | `antioch version` |

All Antioch-managed simulation compute runs in sessions. A session contains one project's
services and is either interactive or background:

- An interactive session belongs to one user. `service exec`, direct service
  access, watch actions, routes, attached scenarios and suites, and Jupyter
  use it.
- A detached scenario or suite uses reusable background sessions. Scenario
  and suite records keep results and progress independently; they do not own
  compute or usage.

Mission Control authoring uses a separate ephemeral workspace. A workspace is
not a session and never runs the simulator.

Ordinary Python can record caller-owned scenarios without a session. It uses
a valid project, a real source file, and existing user or personal-access-token
authority. It does not build images, allocate compute, create a revision, or
start a simulator. Read [caller recording](references/authoring.md#record-from-ordinary-python)
for admission, deadlines, and explicit offline execution.

An interactive run uses the project's one live interactive session; every
command acts on the project of the current directory, so nothing takes a
session selector. For detached work, Antioch reuses compatible background
sessions pinned to the submitted project revision or starts more when needed.

Scenario and suite commands use the project's interactive session and follow
until the run finishes by default. `--no-follow` returns the admitted records
without changing that session or its stream intent. `--detach` selects headless
background compute; `--follow` can wait for either mode. Background fan-out is
automatic within quotas and capacity.

Known retired inputs produce `ANTIOCH-DEP-###` notices and are ignored, not
supported workflows. Remove an old `--parallel` option; background fan-out is
automatic. Remove an old `action: rebuild` watch rule; the whole rule is
dropped and does not sync files or change an image. Image changes need a fresh
session. CLI notices use stderr, including structured notice objects with
`--json`; do not treat them as command results. Unknown inputs still fail.

## Understand the project

A project is rooted at `antioch.yaml`. The file declares one or more services.
The simulator is a role derived from verified engine-image metadata, not a
service name. CLI-managed scenarios, suites, and remote Jupyter require that role. A project
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

The development watch actions are `sync`, `sync+restart`, and `sync+exec`.
They update live files and service processes in an interactive session.
They never change its manifest, revision, images, or containers. Background
submissions build the current YAML independently and never apply watch rules. Load
`references/manifest.md` before editing this file.

## Use image identity

A service declares exactly one published image or local build. For managed compute, Antioch
resolves every active service to an exact content digest and stores the whole
service graph as an immutable project revision. Sessions and managed runs pin that
revision. A local build is skipped only when its content-derived build key
resolves to an immutable registry digest with matching labels.

- An Antioch engine image reference selects the engine and SDK release.
- A registry image is mirrored to an Antioch-owned digest before admission.
- A local build captures the declared Docker context and produces a project
  image with an exact digest.

Before a fresh session or managed run prepares a revision, the CLI checks
authored resource demand against the serving machine catalogs. This precedes
build-context uploads, image builds, and registry copies, including private
images. It is static feasibility, not a live-capacity check or reservation.
Standalone project builds skip this preview; saved revisions and existing
sessions do not repeat fresh preparation. Final admission remains authoritative
and rechecks the frozen revision, image lineage, resource fit, and quotas.

Project source lives at `/workspace/project`. Dockerfile `COPY` places source
in each built image; sync transfers development edits into a live session.
Managed runs do not upload or restore source bundles. A managed rerun uses the pinned images
and parameters, not unbuilt interactive edits.

Keep `pxr`, `omni`, `carb`, `isaacsim`, and `isaaclab*` imports inside
functions or under `if TYPE_CHECKING:`. Local discovery must work without
Isaac installed.

## Work with an interactive session

A project has one live interactive session. `session new` starts a fresh one
and replaces the project's current session; Rome refuses the replacement while
the old session still runs a command unless you pass `--force`, which stops
that work first and skips the terminal confirmation. The default quota is two
interactive sessions across your projects and a separate two background
sessions. A service with no effective command stays idle until exec or Jupyter
uses it. Start a session, or let `service exec` start one only when none is live:

```bash
antioch session new
antioch session list
antioch session status
antioch service exec python src/main.py
antioch service exec --service autonomy -- ros2 topic list
```

`service exec` runs literal argv in the simulator service, or the only active
service when there is no simulator. Use `--service` to select a helper; the
command's first token never selects a service. Use repeated `--profile` options
to activate authored profiles when starting a session. Native commands stream
output and exit status. A command alone creates no run history; its Python code
can record a scenario. Exec forwards stdin and
uses a terminal when local stdin and stdout are terminals; `--tty` and
`--no-tty` override that choice. Non-terminal stdout and stderr stay separate.
Every command acts on the project of the current directory and its one live
session; run it inside the project.

Take direct evidence from existing services:

```bash
antioch service exec --service sim -- nvidia-smi
antioch service logs SERVICE...
antioch service cp sim:/workspace/project/output.png ./output.png
antioch service shell
```

Source enters a session at start and through `antioch service sync` or
`antioch service watch`. Every command runs against the source already in the
session. `service sync` copies files once without restarting or executing
anything; `service watch` runs the continuous declared actions.
`antioch service restart` restarts selected service processes in the same
containers and waits for fresh authored health checks. An idle service stays
idle. A reserved run can block an interactive restart.

For image or dependency changes, update the Dockerfile and run
`antioch session new` again. Save remote files first; nothing copies a live
notebook's memory or temporary files into the fresh session.

Release the session when the work is done:

```bash
antioch session release
```

Releasing a session stops its services and removes its temporary files. A
session can also stop automatically after it has been idle. Background
sessions belong to detached runs; cancel the run instead.

## Use Jupyter

JupyterLab runs in the project's interactive session's simulator service. Kernel and cell
commands use that server's standard Jupyter APIs:

```bash
antioch jupyter lab
antioch jupyter cell '1 + 1'
antioch jupyter cell --stream 'import antioch; antioch.start_simulation()'
antioch jupyter lab --stop
```

Jupyter uses the project's interactive session and never starts one.
`jupyter cell` uses the sole live kernel, starts one when none is live, and needs `--kernel ID`
when several are live. Interrupt `antioch jupyter lab` or run
`antioch jupyter lab --stop` to stop JupyterLab. Load `references/sessions.md`
for the complete workflow. Starting a kernel does not start Isaac. A cell that
starts Isaac inherits Lab's preferred stream request; `--stream` or
`--no-stream` overrides it for that cell only. The flag does not reconfigure a
running simulator. Older kernels refuse explicit flags before executing the
cell; use an authored `SimulationConfig(stream=...)` or start a new session.

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

A CLI-submitted managed run saves its selected inputs and the exact service images before
execution. Later local edits cannot change it.

Read the saved evidence before making a claim:

```bash
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario logs SCENARIO_RUN_ID
antioch scenario download SCENARIO_RUN_ID
antioch suite show SUITE_RUN_ID --json
```

A managed rerun gets a new ID and uses the saved service images and parameters.
Caller-owned records have no revision and do not support managed rerun or live streaming.
Unbuilt interactive edits are not preserved. Images below the supported runtime
floor are refused before allocation; their saved results remain readable. This
does not guarantee the same outcome or timing when scheduling, capacity,
simulator timing, or external assets differ:

```bash
antioch scenario rerun SCENARIO_RUN_ID
antioch suite rerun SUITE_RUN_ID
```

## Diagnose a failure from evidence

Use this order:

1. Read the run result and logs.
2. If the failure is interactive, read the project's session and service state.
3. Confirm the active identity and CLI version.

```bash
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario logs SCENARIO_RUN_ID
antioch session status --json
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
antioch asset list -q "mobile robot" --json
antioch asset show robots/example --json
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
