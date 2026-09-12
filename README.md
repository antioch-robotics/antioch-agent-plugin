# Antioch Agent Plugin

Guidance and research tools for agents working with
[Antioch](https://antioch.com). The plugin helps an agent work in your existing
project, write native Isaac code, run requested evaluations on remote compute,
and inspect their recorded evidence. It does not install a simulator locally
or make an untested simulation correct. Interactive guidance uses immutable
sessions: sync files, restart processes, and start a fresh session for image
changes. Background work builds the same Dockerfile and runs within quotas.

## Capabilities

| Skill | Responsibility |
|---|---|
| `antioch-platform` | Project setup, service graphs, sessions, CLI workflows, assets, runs, suites, and Jupyter |
| `antioch-research` | Search and inspect hosted, versioned vendor documentation and source |
| `isaac-sim-6` | Isaac Sim 6.0.1 physics, USD, assets, sensors, navigation, manipulation, rendering, and datasets |
| `isaac-lab-3` | Isaac Lab 3.0.0-beta2 environments, managers, controllers, and RL integration |
| `scenario-design` | Cases, measured verdicts, artifacts, Rerun telemetry, and review layouts |

Research exposes six tools: `research_search`, `research_artifacts`,
`research_expand`, `research_open`, `research_grep`, and `research_versions`.
Call the version tool to see current coverage; documentation crawls and source
pins are not interchangeable.

The plugin uses your existing Antioch identity. Research queries go to the
hosted index; simulation dispatch and transport use the ordinary CLI/session
interfaces. It bundles no agents that run independently of your harness.

## Install and inspect setup

After the selected deployment and a paired SDK support setup, use one command:

```bash
antioch setup
antioch setup --dry-run
```

Setup installs the deployed SDK and its exact paired plugin for every Claude
Code or Codex client on PATH; a host without an agent gets only the SDK. The SDK
supplies both `antioch` and `antioch-research-mcp`. Keep those programs on the
PATH inherited by the agent. An explicit existing `--python .venv/bin/python`
targets a project environment instead of a global uv tool. Setup never edits a
shell profile.

Production is the default. `ANTIOCH_ENV=staging` selects staging, as it does
for every antioch command. Setup verifies one
release pair, not independent latest SDK and plugin versions. It never signs
in. After installing, it always checks existing sign-in and a real Research
call, and a failed check exits 1 with the exact next step.

Older production metadata can require authentication, and older public SDKs
have no plugin binding. Setup reports this rollout gap before installation.
A fresh `uvx --from antioch-sim antioch setup` works only after a
setup-capable SDK is normally published. Before that, the usable public
bootstrap is SDK-only: `uv tool install --python 3.12 antioch-sim`. It is not
proof of a verified plugin pair. Ask your staging operator for the exact
approved private SDK bootstrap when public PyPI has no setup-capable release.

Restart the agent if its harness needs that to load a plugin. Inspect native
plugin/MCP status and approvals; an installed plugin is not proof that Research
is reachable. Ask the agent to call `research_versions` when that check is
within the task.

Mission Control supplies its own tools and identity. Do not replace its hosted
login or toolchain with a local setup workflow. Setup does not create a project
or start a simulation session.

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

Run `antioch setup` again to update the SDK and plugins to the deployed
release; `antioch setup --dry-run` shows the plan first. Use
`antioch project update --dry-run` and then `antioch project update` in a uv
project to update its active direct SDK dependency, lock and environment, and
literal engine image pins; changed Dockerfile pins build through the normal
revision path unless you pass `--no-build`. Neither command starts or alters a
running session. Optional/group-only SDK selection needs an explicit
interpreter. A dry run makes no installation or configuration changes; native
clients can still write their ordinary inspection logs. Unrelated plugins and
MCP entries are preserved.

Text and JSON distinguish `mode: plan` from `mode: apply`. Components report
`current`, `planned`, or `changed`; the agent map lists every agent on PATH. A
current plugin on a dry run is installed, not newly configured by that read.
Project sync and build actions report `planned`, `completed`, or `skipped`
separately: a completed uv sync need not change files or the SDK, and
`--no-build` cannot prove a build. Read those component results, not an
aggregate configured flag or a command trace.

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
  hide the error. Hosted workspaces use their provided identity.
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
