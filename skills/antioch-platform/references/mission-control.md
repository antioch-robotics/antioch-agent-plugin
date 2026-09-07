# Work in Mission Control

Mission Control is Antioch's browser workbench for live simulation, telemetry,
and hosted development. Its Workspace pane starts one temporary workspace
with prepared projects, a signed-in CLI, JupyterLab, and an agent terminal.
The workspace is separate from the interactive and background sessions that
run simulation services.

The stage automatically plays one active scenario stream. When several
scenarios are streaming, select a session in the Sessions pane. A scenario
reserves its session stream while its simulation process runs.

## Work in the hosted environment

The processes and file system are temporary. Keep source in git or
publish reusable content as an asset.

`ANTIOCH_WORKSPACE_ID` identifies this environment. When it is present:

- use the identity already provided by Mission Control;
- do not run login, logout, or organization-switch workflows;
- treat the file system as temporary;
- save changed source in the user's repository before release.

## Submit simulation work

Mission Control can submit scenarios and suites from its project view. The
workspace is only an authoring client. Simulation work executes in interactive
or background sessions, never in the workspace itself.

The saved run includes resolved service image digests, inputs, outcome, logs,
telemetry, and artifacts. The evidence remains after its compute stops.

## Use the hosted tools

The Agent terminal runs in the workspace. JupyterLab and its files are
hosted in that temporary environment, while the Isaac kernel uses a separate
interactive session. With the local CLI, `antioch jupyter lab` instead
launches JupyterLab in the selected session's simulator service and forwards
its declared route to a local browser. Those notebook files are in the
session, not automatically on the user's computer.

Read submitted work with the ordinary history commands:

```bash
antioch scenario list
antioch scenario show SCENARIO_RUN_ID
antioch suite show SUITE_RUN_ID
```

The webapp links the same scenario and suite evidence.

## Understand usage

Organization usage includes interactive and background session compute only.
Workspace and build compute has no usage ledger. Review session usage on the
webapp's **Usage** page.
