# Mission Control — the hosted workspace seat

Mission Control is Antioch's hosted authoring seat in the webapp: every user
gets exactly one ephemeral cloud workspace with the CLI already signed in,
runnable example projects, hosted JupyterLab, and an agent terminal. An agent
may literally be running inside one. Read this file when the environment
carries `ANTIOCH_WORKSPACE_ID`, when a run's origin shows `Mission Control`,
or when the user works from the webapp instead of a local checkout.

## What a workspace is

- A sandboxed pod claimed from a warm pool — not a GPU machine. It is a
  client of the same GPU fleet the local CLI uses; dispatched work runs on
  assigned machines exactly as it does from a laptop.
- **Ephemeral by design.** An open console keeps it alive; without a
  heartbeat for fifteen minutes the workspace retires — pod, credential, and
  files all go. There is no suspension, no successor, and no surviving
  worktree.
- **Durable work leaves only as dispatched run truth.** Scenario runs, suite
  runs, published assets, and machine usage survive; anything else in the
  workspace filesystem does not. Push source to git, publish results as
  assets, and treat scenario artifacts as the only durable outputs.
- Workspace time is not billed as GPU compute; `antioch machine usage`
  reports machine time only.

## How to recognize one

- `ANTIOCH_WORKSPACE_ID` is set in the environment.
- A `workspace.json` credential exists in the config store and wins over any
  `auth.json` session.
- Example projects live under `~/examples`, ready to dispatch.

## CLI differences inside a workspace

Everything else — scenario, suite, machine, services, assets, project — is
seat-agnostic. The differences:

- **Auth verbs are refused.** `auth login`, `auth logout`, and `auth switch`
  fail with "Mission Control runs as the user who opened it" — the webapp
  owns the account. `antioch auth whoami --json` still reports the identity.
- **The workspace credential is platform-pushed.** It carries no refresh
  token; workspace retirement revokes it. Do not copy it elsewhere.
- **Hosted JupyterLab** is served through the workspace gateway (the hidden
  `--hosted`/`--port` pair on `jupyter lab`); the webapp opens it — do not
  start it by hand. `antioch jupyter stream` exists specifically to put a
  kernel's Isaac GUI on the Mission Control livestream pane.
- The webapp's Launch pane (Run live / Run / Queue) and Activity pane are GUI
  twins of `scenario run`, `--queue`, and the list commands.

## Provenance: `dispatched_from`

Runs and machine assignments record where they were dispatched from. Inside a
workspace the recorded origin is `cloud-workspace`; the CLI and webapp
display and accept the product label `Mission Control`:

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
