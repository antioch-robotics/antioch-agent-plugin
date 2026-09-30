# Antioch Agent Plugin

Project guidance, research, and interactive simulation tools for agents working with
[Antioch](https://antioch.com). The plugin helps an agent work in your existing
project, write native Isaac code, run requested evaluations on remote compute,
and inspect their recorded evidence. It does not install a simulator locally
or make an untested simulation correct. Session images stay fixed: project
files sync automatically, processes restart on request, and image changes
need a fresh session from `antioch session new`. Submitted runs carry a
source bundle and run on sessions Antioch starts within quotas, two at a
time per command unless `--parallel` says otherwise, or with `--session` on
a session of yours.

## Capabilities

| Skill | Responsibility |
|---|---|
| [agentic-simulation](skills/agentic-simulation/SKILL.md) | Workflow entry point: research, design, build, measure, and improve within the requested scope |
| [antioch-platform](skills/antioch-platform/SKILL.md) | Companion programming/cloud model, projects, services, sessions, CLI/YAML, assets, scenarios, and suites |
| [antioch-research](skills/antioch-research/SKILL.md) | Search across hosted vendor documentation and source to choose methods, connect libraries, and ground APIs |
| [isaac-sim-6](skills/isaac-sim-6/SKILL.md) | Isaac Sim 6.0.1 physics, USD, assets, sensors, navigation, manipulation, rendering, and datasets |
| [isaac-lab-3](skills/isaac-lab-3/SKILL.md) | Isaac Lab 3.0.0-beta2 environments, managers, controllers, and RL integration |
| [ros2](skills/ros2/SKILL.md) | ROS 2 against simulations, on the engine's bundled Jazzy or another distro: the Isaac bridge, sim time, tf2, QoS and discovery, Nav2, MoveIt 2, and ros2_control |
| [scenario-design](skills/scenario-design/SKILL.md) | Cases, measured verdicts, artifacts, Rerun telemetry, and review layouts |

Research exposes six tools: `research_search`, `research_artifacts`,
`research_expand`, `research_open`, `research_grep`, and `research_versions`.
Call the version tool to see current coverage; documentation crawls and source
pins are not interchangeable.

The `antioch-jupyter` MCP server (launched by `antioch-jupyter-mcp`) exposes
`jupyter_connect`, `jupyter_execute`, and `jupyter_kernel`. They connect to the
session `jupyter_connect` names with `session`, or else your only running
session of the current project (with several, it refuses and lists them),
select a kernel, run stateful cells, return their output as plain text and
images, and interrupt, restart, or stop the kernel. Each cell runs after the
project's synced files are applied. Execution has no implicit viewport capture.
Project files, builds, native scripts, scenarios, suites, and saved evidence
use the existing SDK and CLI.

An authoring request includes a runnable local project: environment, source,
`antioch.yaml`, and a service image with the required runtime. The manifest's
`sync` mappings deliver source; the image supplies the environment. An existing project
is repaired in place. Run a project Python file with
`antioch service exec -- python src/main.py`, the same command that runs
anything else in a service. A plain script needs no
scenario decorator, but it still needs project configuration for remote dispatch. A question alone
does not authorize files, installs, or compute.

The plugin uses your existing Antioch identity. Research queries go to the
hosted index; Jupyter transport uses the ordinary CLI/session interfaces. It
bundles no agents that run independently of your harness.

## How the skills are maintained

Agentic simulation owns the workflow; the platform skill owns CLI/YAML and
compute concepts. Scenario design owns Python evaluation APIs; the Isaac skills
supply native simulator knowledge, and the ROS 2 skill the robot stack that
drives it. Research connects these skills to evidence.
Task-specific links form a navigable reference graph. Each entrypoint routes to detailed guides; those guides link back to their owner and across to the next task. For example, an agent can move from asset discovery to USD placement, robot control, scenario checks, and recorded telemetry without reading unrelated material. CLI operations and the public Python client share the platform guides; the client guide explains automation and recovery rather than duplicating shell commands.

Isaac Sim references preserve NVIDIA's domain guidance, examples, and topic
structure with a small Antioch adaptation layer: remote execution, startup,
import safety, extension configuration, and evidence-backed corrections.
Each reference links its upstream commit. We check that guidance against the
shipped engine version rather than tracking `develop` blindly. We simplify
duplicated instructions, not the domain knowledge needed to build a simulation.

Linked upstream helper scripts are source examples, not installed commands.
Local installation and upstream agent-orchestration instructions are outside
this plugin. [NOTICE](NOTICE) records provenance and licensing.

## Install and inspect setup

Install the [Antioch SDK](https://console.preview.antioch.com/docs/quickstart/install-the-sdk)
in your project's Python 3.12 virtual environment, then activate it and sign in.
From the project directory, preview and apply plugin setup:

```bash
antioch setup --dry-run
antioch setup
```

Setup installs the deployment's plugin for every Codex or Claude Code client
on PATH, then checks your sign-in and Research connection. If a check fails,
follow the reported next step. Setup does not install or upgrade the SDK, sign
you in, or start compute.

Launch your agent from the activated environment so it can find `antioch`,
`antioch-research-mcp`, and `antioch-jupyter-mcp`. Restart an already-running
agent after an update. Check its plugin and MCP status, then ask it to call
`research_versions` to confirm the Research connection.

## Use it

Start from the project directory and give the agent a concrete task:

> Inspect this autonomy stack and design an obstacle-avoidance scenario.
> Reuse the robot and services. Define measurable checks. Run one
> representative case, then explain the saved result and any failed checks.

For a read-only task, say so:

> Check this Isaac API against the runtime version and explain the failure.
> Do not dispatch or change code.

An API lookup can verify an interface; only a relevant runtime check can
verify physical behavior. Review the agent's reported tests, run IDs, artifacts,
failures, and unrun checks. The plugin tells agents to preserve that distinction
and to keep failed samples as diagnostic evidence.

## Updates and removal

Run `antioch setup --dry-run` to preview a plugin update, then
`antioch setup` to apply it. Upgrade the SDK and engine image yourself as
described in [Upgrade the SDK](https://console.preview.antioch.com/docs/quickstart/install-the-sdk#upgrade-the-sdk).

To remove this plugin, use `claude plugin uninstall antioch@antioch` or
`codex plugin remove antioch@antioch`. Remove its marketplace only if no
remaining installation needs it. The separate SDK installation and Antioch
login remain until explicitly removed.

## Troubleshooting

- **Executable missing:** verify PATH in the environment that launches the
  agent. Activating a virtual environment in a different terminal does not
  change an already-running agent.
- **Plugin present, tools absent:** inspect the harness's plugin/MCP status
  and pending approvals. A CLI list is not a successful research call;
  ask the agent to call `research_versions`.
- **Research authentication error:** inspect `antioch auth whoami` and follow
  the returned login instruction. Do not switch organization/deployment to
  hide the error.
- **Research service unavailable:** report the returned error. Official
  source at the matching pin or checked-in types can provide a labeled
  fallback, but are not live verification.
- **Stale explicit MCP configuration:** inspect the configured executable
  and plugin source. Remove an entry only after confirming it is obsolete
  and obtaining permission; its mere presence does not make it wrong.
- **Protocol error with an older SDK:** check the installed version and
  server output, then update through the owning package manager. Do not
  assume every connection failure has the same cause.

## License

Apache-2.0. Isaac Sim guidance includes material adapted from NVIDIA's
Apache-2.0 skills. [NOTICE](NOTICE) records attribution. Indexed research
sources retain their own licenses.
