# Sessions and direct execution

Sessions supply compute; scenarios and suites supply recorded evaluations.
The same scenario can run interactively during development and in background
for repeatable evaluation.

## Choose the execution mode

| Scenario/suite options | Compute | CLI returns |
|---|---|---|
| Default | Project's interactive session | After terminal result |
| `--no-follow` | Same interactive session | After admission |
| `--detach` | Headless background sessions | After admission |
| `--detach --follow` | Same background execution | After terminal result |

Interactive compute supports scripts, shells, sync, routes, Jupyter, and
serial recorded runs. It remains available after a run. Background dispatch
builds a frozen revision independently of the live session; suite members can
fan out within capacity and quotas. Background compute retires automatically.
It has no GUI stream, source sync, or Jupyter attachment. Control its work with
scenario/suite follow and cancel commands, not interactive service commands.

## Start or reuse

```bash
antioch session status --json
antioch session list --json
antioch service exec python src/main.py
```

Commands select the current user's project in the working directory, not a session ID.
Exec reuses its interactive session or creates one when absent. It defaults
to the simulator service, or the sole active service. Put `--service` and
other Antioch options before the command; argv is passed literally.

`antioch session new` builds a fresh revision and replaces the project's
interactive session. It waits for readiness and copies initial source from
sync rules, without running watch restart/exec actions. Save remote files
first. Active work prevents replacement unless authorized cancellation uses
`--force`. Capacity or quota refusals do not authorize stopping other work.

Exec streams output and exit status, forwards terminal input when applicable,
and Ctrl-C stops the exact process, not the session. It requests the GUI stream
by default; `--no-stream` leaves the session's single producer slot free.

## Edit and inspect

```bash
antioch service sync
antioch service watch
antioch service restart
antioch service ps --json
antioch service logs SERVICE...
antioch service shell
antioch service cp sim:/workspace/project/output.png ./output.png
```

Exec uses files already in the session; it never copies later edits.
Sync copies once. Watch applies declared `sync`, `sync+restart`, and
`sync+exec` rules. Restart changes service processes in their existing
containers and waits for fresh health checks. An idle service without a command
has nothing to restart. A reserved run can prevent restart.

A restart timeout does not undo launch. Inspect state before starting another
restart, which would reset the process again. An unhealthy but running session
can still support diagnosis; a stopped, failed, or revoked one needs replacement.

Session images, revision, manifest, and containers do not change in place.
Image or dependency edits need `antioch session new`.
Logs, shell, and copy require an existing session. Copy paths must be inside
`/workspace/project`; use an authorized service command to move other outputs
there before downloading.

## Named routes

Declare ports and direction in the manifest, then keep the forwarder running:

```bash
antioch service ports --bind sim.viewer=127.0.0.1:8080
antioch service ports --serve
```

For client-to-service routes, bind chooses the local listener. For
service-to-client routes, it chooses the required client destination: the
service connects to its declared local port and the bridge forwards to that
destination. `--clear sim.viewer` removes a binding. Stopping the forwarder
closes connections, not compute.

## JupyterLab

```bash
antioch jupyter lab
antioch jupyter cell 'print("ok")'
antioch jupyter cell --stream 'import antioch; antioch.start_simulation()'
```

Jupyter needs an existing interactive session with a simulator service.
It does not allocate compute. Cell execution selects the sole kernel or starts
one if absent; use `--kernel KERNEL_ID` when several exist. Starting a kernel
does not start Isaac.

Cell `--stream`/`--no-stream` applies during simulator startup, not to an
already-running app. Interrupting Lab or `antioch jupyter lab --stop` stops
Lab and discards its kernels; use the owning client and preserve other work.
Notebook files are remote. Agent kernel controls and explicit images are in
[the Jupyter MCP guide](../../agentic-simulation/references/jupyter.md);
recording a notebook is in
[caller recording](../../scenario-design/references/recording.md).

## Release

```bash
antioch session release
```

Save source and outputs first. Release starts asynchronous teardown and refuses
active work unless authorized `--force` cancellation is supplied. Closing a
process, kernel, or adapter does not release compute. Background work is
cancelled through its scenario/suite, not this command.

Interactive sessions stop after 15 minutes without observed work. Managed
commands, open shells, executing notebook cells, and active recorded runs
keep them active; idle notebook servers, persistent service entrypoints,
streams, status reads, and open console tabs do not. Raw host SSH and untracked
notebook servers are outside this activity boundary. Missing runtime activity
reports defer cleanup without extending the deadline; older sessions need
replacement to use this policy. The console Usage page reports organization
session compute.
