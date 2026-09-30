---
name: antioch-platform
version: "1.5.42"
description: Use for Antioch projects, CLI and Python client commands, antioch.yaml, sessions, services, builds, source sync, streams, assets, scenarios, suites, run history, and authentication. Explains the platform model and routes to detailed references; native engine code belongs to the Isaac skills and ROS 2 stacks to ros2.
---

# Antioch

Antioch runs your Python on cloud GPUs. The local SDK supplies the CLI,
Python APIs, and editor types; remote images contain the simulator. No local
Isaac installation or GPU is needed. Keep native engine APIs in the project.

## Find the project

Use the nearest `antioch.yaml` in the current directory or its parents.
Preserve its identity, engine, dependencies, and user changes. Run commands
from that root with its Python environment active; launch the agent there too
so the CLI and MCP adapters use the same SDK. Installed `--help` and SDK
models are the authority for options. `antioch version --json` reports
component versions and the deployment; `ANTIOCH_TRACE=1` explains command timing.

Work within the request. A question needs no compute; an access error does
not authorize changing identity. Continue useful local work when remote
access is blocked and report which checks remain unrun.

## Choose how to work

A **project** is a source directory with `antioch.yaml`, Python dependencies, and image recipes. Its **services** are containers for the simulator and supporting software. Antioch builds their images remotely and saves them for reuse across the organization. A **revision** freezes the images and manifest. A **session** runs that revision on temporary compute. Organization identity scopes shared assets and run history; sessions belong to the person using them.

| Need | Start here |
|---|---|
| Run a script in an existing session | `antioch service exec -- python src/main.py` |
| Iterate without restarting the simulator | [Jupyter kernel](../agentic-simulation/references/jupyter.md) |
| Record inputs, checks, results, and artifacts | `antioch scenario run NAME`; [scenario design](../scenario-design/SKILL.md) |
| Run a named selection of scenarios and cases | `antioch suite run NAME`; [suite YAML](references/suites.md) |

`antioch session new` builds when needed and adds a session. Direct commands
never allocate compute: use `--session SESSION_ID`, `--run RUN_ID` for a
session a run used while that session remains live, or your only running session of this project that you
started. Ambiguous selection is refused. Scenario and suite dispatch can
obtain compute or target a session explicitly. Without a target, Antioch reuses sessions it started for your runs, never one you started yourself. Both count toward the same session quota. See [sessions](references/sessions.md) for selection, concurrency, and release.

A **scenario** is a typed Python evaluation; a **case** supplies parameter
values. A **suite** selects scenarios and cases in YAML. Each execution
creates run records and uploaded evidence that survive the session.
A plain script creates no run history unless it calls a scenario.

## Keep code and compute in sync

Image or dependency changes need a new session. Source edits use the
manifest's `sync` mappings to `/workspace/project`: scripts, shells, and
notebooks copy local files before starting and keep syncing while attached. Submitted runs
carry a frozen source bundle and apply it when they start; reruns reuse the
saved images and bundle. A command can overwrite files even while a run is
executing, so avoid concurrent edits when preserving its inputs matters.

Keep simulator imports inside functions or `TYPE_CHECKING` blocks so local
discovery works without Isaac. Closing Python or an MCP connection does not
release compute; save remote outputs before releasing a session you own.

## Capability guides

Load the guide for the operation you need:

| Task | Guide |
|---|---|
| `init`, `setup`, `version`: install tools, create projects, inspect versions, change images | [Environment](references/environment.md) |
| Services, resources, routes, source mappings | [Manifest](references/manifest.md) |
| `session`, `service`, `jupyter`: inspect/use/release compute, sync files, forward ports, open streams | [Sessions](references/sessions.md) |
| `scenario`: collect, dispatch, filter, download, cancel, delete, or rerun evaluations | [Scenarios](references/scenarios.md) |
| `suite`: collect, run, summarize, compare, cancel, delete, or repeat groups | [Suites](references/suites.md) |
| `asset`: find, load, publish, verify, or repair reusable content | [Assets](references/assets.md) |
| `auth`, including `auth registry`: identity and image access | [Authentication](references/auth.md) |
| ROS bridge and cross-service traffic | [ROS 2](references/ros2.md) |
| ROS 2 nodes, tf2, Nav2, MoveIt 2, ros2_control against the simulator | [ROS 2](../ros2/SKILL.md) |
| Script startup, native state, and rendering configuration | [Simulation code](references/simulation-code.md) |
| Automate sessions, commands, submissions, and history in Python | [Python client](references/python-client.md) |
| Diagnose a failed workflow | [Troubleshooting](references/troubleshooting.md) |

Use [agentic simulation](../agentic-simulation/SKILL.md) for the experiment
loop, [research](../antioch-research/SKILL.md) for native APIs,
[Isaac Sim](../isaac-sim-6/SKILL.md) or [Isaac Lab](../isaac-lab-3/SKILL.md)
for engine code, and [scenario design](../scenario-design/SKILL.md) for
measured verdicts and telemetry.
