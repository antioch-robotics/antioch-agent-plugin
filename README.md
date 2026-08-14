# Antioch Agent Plugin

[Antioch](https://antioch.com) is the simulation platform for robotics and
autonomy teams. The official Antioch Agent Plugin turns Codex or Claude Code
into an expert simulation engineer that works from your existing repository.

Your agent can understand your robotics stack, research exact simulation APIs,
write native Isaac code, and build repeatable Antioch scenarios. It can connect
your autonomy services, robot and sensor models, and simulation assets; run
evaluations across Antioch GPU machines; and use the recorded results to improve
the implementation.

## What your agent can do

- **Build in your project.** Inspect `antioch.yaml`, work with your existing
  services and source, and author simulations without cloning the project onto
  a remote VM.
- **Research the simulation stack.** Search versioned documentation and pinned
  source for Isaac Sim, Isaac Lab, Omniverse and Kit, OpenUSD, PhysX, Newton,
  Warp, and other supported libraries through Antioch Research.
- **Run and evaluate simulations.** Start interactive work, use Jupyter, run
  scenarios and suites, and fan evaluations across several Antioch machines.
- **Reason from evidence.** Inspect outcomes, checks, logs, telemetry,
  artifacts, and visualizations, then use that evidence for the next iteration.

The plugin does not install Isaac on your computer. Your code remains ordinary
Python, while Antioch runs the simulator on a remote GPU machine.

## Install Antioch

The plugin uses the Antioch CLI to run simulations and the
`antioch-research-mcp` executable to research the simulation stack. The easiest
way to install both is with [`uv`](https://docs.astral.sh/uv/getting-started/installation/). After installing `uv`, run:

```bash
uv tool install --python 3.12 antioch-sim
uv tool update-shell
```

Start a new shell if `antioch` is not yet on your path. Then sign in and confirm
the active account:

```bash
antioch auth login
antioch auth whoami
```

If your project already installs `antioch-sim`, activate its virtual
environment before you start the agent and skip the global tool installation.
A project that authors simulations must also select one supported engine extra.
The [Antioch SDK guide](https://console.poc.antioch.com/docs/quickstart/install-the-sdk)
covers project setup, engine selection, and your first run.

## Install the plugin

The commands below pin plugin version `v0.2.12`. To install the moving `main`
branch instead, remove `#v0.2.12` from the Claude Code URL or remove
`--ref v0.2.12` from the Codex command. A floating install can change without
notice.

### Claude Code

```bash
claude plugin marketplace add antioch-robotics/antioch-agent-plugin.git#v0.2.12
claude plugin install antioch@antioch
```

Confirm that the plugin and Antioch Research are available:

```bash
claude plugin details antioch@antioch
claude mcp list
```

Claude Code can show a new project MCP server as pending until you approve it.

### Codex

```bash
codex plugin marketplace add antioch-robotics/antioch-agent-plugin.git --ref v0.2.12
codex plugin add antioch@antioch
```

Confirm that the plugin is enabled and `antioch-research` appears in the MCP
list:

```bash
codex plugin list --json
codex mcp list --json
```

Restart the agent after installation.

## Put the agent to work

Start Codex or Claude Code from an Antioch project with its environment active. On macOS or Linux:

```bash
cd /path/to/my-sim
source .venv/bin/activate
codex
```

On Windows PowerShell:

```powershell
cd C:\path\to\my-sim
.venv\Scripts\Activate.ps1
codex
```

Run `claude` instead of `codex` for Claude Code. Then give the agent a concrete
robotics objective. For example:

> Inspect this robotics stack and build an Antioch scenario for its obstacle
> avoidance behavior. Reuse the existing autonomy services, robot model, and
> sensors. Parameterize the obstacle layout and speed, define clear checks, run
> representative cases across several machines, and use the saved telemetry and
> artifacts to explain any failures.

You can also ask the agent to:

- “Research the exact Isaac API this change needs, implement it, and validate
  the result on Antioch.”
- “Turn this working simulation into a parameterized scenario and a regression
  suite.”
- “Compare these suite runs and explain the failures from their recorded
  evidence.”

The first research request can prompt you to approve the MCP server. A healthy
installation can call `research_versions` and return the indexed corpus table.

## How it works

The plugin gives the agent focused guidance for the Antioch platform, scenario
design, Isaac Sim, and Isaac Lab. It also connects the agent to **Antioch
Research**, which grounds API and implementation questions in versioned
documentation and source instead of model memory.

The agent uses your existing Antioch identity and works through the same local
CLI as you. Simulation traffic does not pass through the research connection:
the CLI connects directly to your Antioch machine, while the research server
only queries Antioch's documentation and source index.

Review the [complete agent guide](https://console.poc.antioch.com/docs/agents/work-with-agents)
for project setup, example workflows, and guidance for reviewing autonomous
simulation work.

## Update

If you installed the Antioch tools globally with `uv tool`, upgrade them first:

```bash
uv tool upgrade antioch-sim
```

If the plugin uses an Antioch project environment instead, update
`antioch-sim` with that project's package manager.

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

Remove the separately installed Antioch tools only if no other project uses
them:

```bash
uv tool uninstall antioch-sim
```

Run `antioch auth logout` if you also want to remove the local Antioch login.

## Troubleshooting

- **`antioch-research-mcp` is not found:** run `uv tool list` and confirm that
  `antioch-sim` appears. Then run `uv tool update-shell`, start a new shell, and
  check `claude mcp list` or `codex mcp list --json` again.
- **Antioch Research reports that authentication is required:** run
  `antioch auth login`, then retry the request.
- **The plugin is installed but its guidance is absent:** restart the agent and
  check `claude plugin details antioch@antioch` or
  `codex plugin list --json`.
- **The MCP server is not listed:** confirm that the plugin is enabled, then
  reinstall it from the refreshed `antioch` marketplace.

## License and attribution

The plugin is licensed under Apache-2.0. Some Isaac Sim guidance is adapted
from NVIDIA's Apache-2.0 Isaac Sim skills. See [`NOTICE`](NOTICE) for the exact
attribution and upstream skill list.
