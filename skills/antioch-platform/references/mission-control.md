# Mission Control — hosted authoring

Mission Control is Antioch's hosted authoring environment in the webapp. It
provides a signed-in CLI, runnable example projects, JupyterLab, and an agent
terminal. Read this file when `ANTIOCH_WORKSPACE_ID` is set, when a run's
origin shows `Mission Control`, or when the user works from the webapp instead
of a local checkout.

## What a workspace is

- A workspace is an isolated authoring environment, not a GPU machine. It
  dispatches simulation work to assigned machines just like the local CLI.
- **Ephemeral by design.** The workspace and its local files disappear after
  it retires. Do not use its filesystem as durable storage.
- **Save work before the workspace retires.** Scenario runs, suite
  runs, published assets, and machine usage survive; anything else in the
  workspace filesystem does not. Push source to git, publish results as
  assets, and treat scenario artifacts as the only durable outputs.
- Workspace time is not billed as GPU compute; `antioch machine usage`
  reports machine time only.

## How to recognize Mission Control

- `ANTIOCH_WORKSPACE_ID` is set in the environment.
- Example projects live under `~/examples`, ready to dispatch.

## CLI differences inside a workspace

Scenarios, suites, machines, services, assets, and projects work through the
same CLI. The differences are:

- **Auth verbs are refused.** `auth login`, `auth logout`, and `auth switch`
  fail with "Mission Control runs as the user who opened it" — the webapp
  owns the account. `antioch auth whoami --json` still reports the identity.
- **Hosted JupyterLab** is served through the workspace gateway (the hidden
  `--hosted`/`--port` pair on `jupyter lab`); the webapp opens it — do not
  start it by hand. `antioch jupyter stream` exists specifically to put a
  kernel's Isaac GUI on the Mission Control livestream pane.
- The webapp's Launch pane (Run live / Run / Queue) and Activity pane are GUI
  twins of `scenario run`, `--queue`, and the list commands.

## Provenance: `dispatched_from`

Runs and machine assignments record where they were dispatched from. The CLI
and webapp display and accept the product label `Mission Control`:

```bash
antioch scenario list --dispatched-from 'Mission Control' --json
antioch scenario suggest dispatched_from --json
```

Machine rows carry the same field. When a workspace retires, Antioch drains
exactly the work it dispatched — another reason durable results must already
be in run records or assets.

## Working discipline for agents in a workspace

- Assume the filesystem can vanish after fifteen idle minutes. Commit and
  push early; dispatch scenarios instead of leaving results in local files.
- Continue an existing notebook session with
  `antioch jupyter cell --kernel KERNEL_ID 'print("hi")'` rather than
  starting a second kernel.
- The auth remedies in `auth.md` that involve `auth login` do not apply here;
  a credential problem inside a workspace means the workspace itself expired
  — reopen it from the webapp.
