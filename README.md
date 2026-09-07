# Antioch Agent Plugin

Guidance and research tools for agents working with
[Antioch](https://antioch.com). The plugin helps an agent work in your existing
project, write native Isaac code, run requested evaluations on remote compute,
and inspect their recorded evidence. It does not install a simulator locally
or make an untested simulation correct.

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

## Prerequisites

Both `antioch` and `antioch-research-mcp` must be on the PATH inherited by the
agent. One installation provides both:

```bash
uv tool install --python 3.12 antioch-sim
uv tool update-shell
```

Open a new terminal, then verify the programs and sign in if needed:

```bash
which antioch antioch-research-mcp
antioch auth login
antioch auth whoami
```

If your project already installs the SDK, activate its environment before
starting the agent instead of installing another copy:

```bash
source .venv/bin/activate
which antioch antioch-research-mcp
```

These are shell examples for Linux/macOS. Use your shell's equivalent on other
systems. Mission Control supplies its own identity and tools; do not replace
that hosted login with a local login workflow.

The plugin does not create an Antioch project by itself. See the
[SDK setup guide](https://console.preview.antioch.com/docs/quickstart/install-the-sdk).
A manifest needs services, but no service must be named `sim`. The simulator
is selected from the engine-backed services, with an explicit runner marker
when needed.

## Install in your harness

### Claude Code

```bash
claude plugin marketplace add antioch-robotics/antioch-agent-plugin
claude plugin install antioch@antioch
claude plugin details antioch@antioch
claude mcp list
```

### Codex

```bash
codex plugin marketplace add antioch-robotics/antioch-agent-plugin
codex plugin add antioch@antioch
codex plugin list --json
codex mcp list --json
```

These commands install from the public repository. For reproducible
installation, choose an existing tag from the
[public releases](https://github.com/antioch-robotics/antioch-agent-plugin/releases).
Claude accepts `owner/repo#TAG` as the marketplace source; Codex accepts
`--ref TAG` on marketplace add. A version in the development monorepo is not
necessarily published.

Restart the agent if the harness requires it to load new plugins. Tool approval
and plugin visibility depend on the harness and its settings. For other
Agent Skills-compatible harnesses, load this package's canonical `skills/`
tree and register `.mcp.json` through that harness's supported mechanism.

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

Upgrade an unpinned tool installation with `uv tool upgrade antioch-sim`.
For project-owned dependencies, use the project's package manager instead.
Check `antioch --version` in the same environment that starts the agent.

Refresh the marketplace and update/reinstall the selected plugin through the
harness's plugin commands. Inspect `--help` for that installed harness version.
If the marketplace was pinned, select the new released tag explicitly.
Do not remove unrelated plugins or MCP entries during an upgrade.

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
