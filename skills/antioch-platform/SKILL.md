---
name: antioch-platform
version: "1.5.7"
description: >-
  Explains Antioch's programming and cloud model and guides projects, CLI/YAML/SDK use, sessions, assets, scenarios, suites, results, and authentication. Load whenever Antioch is mentioned (including Anticoh), antioch.yaml or Antioch imports are present, or the conversation concerns Antioch. Supplies platform concepts alongside the agentic-simulation workflow entry point; routes native programming to the Isaac skills, cross-library research to antioch-research, and evaluation design to scenario-design.
---

# Antioch

Antioch runs user-owned Python on cloud GPUs. The development environment
is the authoring client. Its SDK provides Python APIs, the CLI,
and editor types; remote engine images contain Isaac and runtime dependencies.
**No local Isaac installation or GPU is needed.** Users keep native simulator
APIs in their code: Antioch abstracts infrastructure, not code.

See [agentic simulation](../agentic-simulation/SKILL.md) for the workflow and
[research](../antioch-research/SKILL.md) to choose and connect native methods.
The [project environment](references/environment.md) supplies engine images
and dependencies for custom code as well as standard examples.

## Product model

- **Compute:** projects declare services in `antioch.yaml`. Antioch builds
  their images remotely and runs them in GPU sessions.
- **Simulation:** native engines supply physics, rendering, sensors, and
  robots. The asset catalog holds reusable, versioned content.
- **Evaluation:** scenarios and suites retain inputs, checks, results,
  logs, artifacts, and telemetry for comparison.
- **Agent tools:** skills, research, Jupyter, the CLI, and SDK connect
  authoring, execution, inspection, and improvement.

A **project** is a source directory with `antioch.yaml`, dependencies, and
image recipes. Its **services** are container workloads: a simulator and any
supporting processes. A **revision** freezes the service images and manifest.

A **session** is temporary remote compute running that revision:

- **Interactive:** one live session per user/project for scripts, shells,
  source sync, Jupyter, and serial scenario or suite execution. Reuse it while
  developing; release it when finished.
- **Background:** detached scenario or suite execution against frozen images.
  Antioch schedules work within capacity and quotas and retires its compute
  after execution. It does not use the project's interactive session.

A **scenario** is a typed Python evaluation with inputs, checks, results, and
evidence. A **case** names parameter values. A **suite** selects scenarios and
cases in YAML. Each execution creates new **run records**; those records and
uploaded artifacts survive session retirement. A raw script creates no run
history unless it explicitly records a scenario.

A Jupyter **kernel** is a Python process whose state persists between cells
inside an interactive session.

Organization identity scopes shared assets and run history. Rome is the
control plane; source and live process, Jupyter, and media traffic travel
directly between the client and session. Save source and evidence before
remote compute retires.

## Start with the owning project

Inspect the current directory and parents up to the repository or known
authoring root. If needed, search shallow task directories for
`antioch.yaml`, excluding caches and unrelated repositories. Use the nearest
owning project; ask when several fit.

A runnable simulation includes its environment, manifest, image recipe, and
source. For a missing or incomplete project, follow
[setup and repair](references/environment.md) and create the missing pieces.
A plain Python script needs project configuration, but no scenario or suite.
Preserve existing project IDs, engine choices, dependencies, and user work.

Run project commands from that root with its environment. The global CLI,
project SDK, and running MCP may use different versions. Installed command
`--help` and SDK models are the authority for options; use `--json` for
structured records rather than parsing display tables.

Match actions to the request. Questions and reviews do not authorize edits,
installs, login, builds, or dispatch. If remote access fails, complete
authorized local authoring and validation; report remote checks as unrun.
Do not change identity or install local Isaac to repair a lookup.

## Authoring and execution

Use ordinary Python for simulation logic, YAML for services and suite
selection, and the CLI for builds, sessions, dispatch, and history.
[Scenario design](../scenario-design/SKILL.md) owns the Python authoring API:
decorators, typed parameters, cases, checks, results, artifacts, and recording existing code. Load the
relevant Isaac skill before writing native simulator code.

| Deliverable | Path |
|---|---|
| Plain script | `antioch service exec python src/main.py` |
| Stateful exploration | Interactive session and [agentic simulation](../agentic-simulation/SKILL.md) |
| Recorded test | `antioch scenario run --scenario NAME` |
| Parameterized evaluation | `antioch suite run NAME` |

Scenario and suite dispatch uses interactive compute by default. `--detach`
selects background compute; `--follow` and `--no-follow` control only whether
the CLI waits. They do not change where work runs. See
[session modes and lifecycle](references/sessions.md).

`service exec` can build and allocate compute when no interactive session
exists; it is not a local check. A project has one live interactive session.
`session new` replaces it; source sync changes files in the existing
session, while image/dependency changes need a new one.

Project files run at `/workspace/project`. Session creation copies initial
source; later exec calls do not sync edits. Background runs need source baked
into their images. A rerun uses saved images and inputs, not unbuilt edits.

Keep `pxr`, `omni`, `carb`, `isaacsim`, and `isaaclab*` imports inside
functions or `TYPE_CHECKING` blocks so local discovery works without Isaac.
Read completed records and artifacts before reporting success. Closing Python,
Kit, or an MCP connection does not release session compute.

## Capability guides

Load the guide for the task, then follow its links as questions arise:

| Task | Guide |
|---|---|
| Install/update tools, create or repair projects | [Environment](references/environment.md) |
| Services, images, resources, routes, watch, profiles | [Manifest](references/manifest.md) |
| Interactive/background modes, execution, sync, Jupyter, release | [Sessions](references/sessions.md) |
| Scenario dispatch, filters, logs, artifacts, reruns | [Scenarios](references/scenarios.md) |
| Suite selection, execution, comparison | [Suites](references/suites.md) |
| Antioch catalog and native Isaac assets: find, load, measure, publish | [Assets](references/assets.md) |
| Existing identity and login workflows | [Authentication](references/auth.md) |
| ROS bridge and cross-service traffic | [ROS 2](references/ros2.md) |
| Build, inspect, evaluate, and repair simulations | [Agentic simulation](../agentic-simulation/SKILL.md) |
| Native scene, physics, sensor, and robot code | [Isaac Sim](../isaac-sim-6/SKILL.md) |
| Environments and RL training | [Isaac Lab](../isaac-lab-3/SKILL.md) |
| Cross-library methods, APIs, examples, and source | [Research](../antioch-research/SKILL.md) |
| Python scenario authoring, cases, checks, results, telemetry, recording | [Scenario design](../scenario-design/SKILL.md) |
