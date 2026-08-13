# Antioch Agent Plugin

The official Antioch plugin helps Claude Code and Codex design, run, and
analyze robotics simulations. It provides five simulation-engineering skills
and one Antioch Research MCP server grounded in versioned documentation and
source.

The public preview uses the Antioch staging service.

## What is included

| Skill | Use it for |
|---|---|
| `antioch-platform` | Antioch projects, authentication, machines, services, scenarios, suites, assets, and CLI JSON |
| `isaac-sim-6` | Isaac Sim 6.0.1 authoring, physics, sensors, navigation, manipulation, rendering, and synthetic data |
| `isaac-lab-3` | Isaac Lab 3.0.0-beta2 environments and its version-specific APIs |
| `scenario-design` | Repeatable cases, pass/fail checks, results, artifacts, telemetry, and Rerun layouts |
| `antioch-research` | Grounded research across indexed Isaac, Omniverse, USD, PhysX, Warp, and robotics sources |

The `antioch-research` MCP server comes from the `antioch-sim` Python package.
The plugin does not install Isaac locally. Simulation runs on an Antioch GPU
machine.

## Prerequisites

Install [uv](https://docs.astral.sh/uv/getting-started/installation/) and the
Antioch command-line tools:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv tool install antioch-sim
uv tool update-shell
```

Start a new shell if `antioch` is not yet on `PATH`. Then sign in to staging:

```bash
ANTIOCH_ENV=staging antioch auth login
ANTIOCH_ENV=staging antioch auth whoami
```

A project that authors simulations must also declare one engine extra. See the
[`antioch-sim` installation guide](https://pypi.org/project/antioch-sim/) for
the project setup and first run.

## Install for Claude Code

```bash
claude plugin marketplace add antioch-robotics/antioch-agent-plugin
claude plugin install antioch@antioch
```

Verify the installation:

```bash
claude plugin details antioch@antioch
claude mcp list
```

The details must show five skills and one `antioch-research` MCP server. Claude
Code can show a new project MCP server as pending until you approve it.

## Install for Codex

```bash
codex plugin marketplace add antioch-robotics/antioch-agent-plugin
codex plugin add antioch@antioch
```

Verify the installation:

```bash
codex plugin list --json
codex mcp list --json
```

The plugin must be enabled, and `antioch-research` must be in the MCP list.

## First use

Restart the agent after installation. In the new session, ask the agent to call
`research_versions`. A healthy installation returns the indexed corpus table.
Then try one of these requests:

- “Research this Isaac Sim API, then explain how to use it in Antioch.”
- “Create an Antioch scenario for this robotics behavior.”
- “Compare these suite runs and explain what changed.”

The research server uses the same staging profile and login as the Antioch CLI.
It sends no simulator traffic through the MCP connection.

## Update

Update the SDK tools first:

```bash
uv tool upgrade antioch-sim
```

Then refresh the marketplace and plugin.

Claude Code:

```bash
claude plugin marketplace update antioch
claude plugin update antioch@antioch
```

Codex:

```bash
codex plugin marketplace upgrade antioch
codex plugin add antioch@antioch
```

Restart the agent after an update.

## Remove

Claude Code:

```bash
claude plugin uninstall antioch@antioch
claude plugin marketplace remove antioch
```

Codex:

```bash
codex plugin remove antioch@antioch
codex plugin marketplace remove antioch
```

Remove the separately installed Antioch command-line tools only if no other
project uses them:

```bash
uv tool uninstall antioch-sim
```

Use `antioch auth logout` to remove the local Antioch login.

## Troubleshooting

- **`antioch-research-mcp` is not found:** run `uv tool update-shell`, start a
  new shell, and confirm that `command -v antioch-research-mcp` prints a path.
- **The research server reports that authentication is required:** run
  `ANTIOCH_ENV=staging antioch auth login`, then restart the agent.
- **The plugin is installed but its skills are absent:** restart the agent and
  check `claude plugin details antioch@antioch` or
  `codex plugin list --json`.
- **The MCP server is not listed:** confirm that the plugin is enabled, then
  reinstall it from the refreshed `antioch` marketplace.

## License and attribution

The plugin is Apache-2.0 licensed. Some Isaac Sim skill content is adapted from
NVIDIA's Apache-2.0 Isaac Sim skills. See [`NOTICE`](NOTICE) for the exact
attribution and upstream skill list.
