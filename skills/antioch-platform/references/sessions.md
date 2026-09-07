# Work with sessions

A session runs one project's services on managed compute. It is
`interactive` or `background`. Interactive sessions support `services exec`,
direct service access, development watch actions, routes, attached scenarios
and suites, and Jupyter. Background sessions are reusable compute pinned to a
project revision for unattended scenario and suite work.

Only interactive sessions can be selected for direct work. Read-only status
and logs can inspect either kind.

## Start and select a session

```bash
antioch session start
antioch session list
antioch session status --session SESSION
```

`session start` starts or reuses a compatible interactive session. Use
profiles when optional supporting services are needed; inspect
`antioch session start --help` for the current syntax.

A user holds one live interactive session at a time by default. Stop a
session you do not need before starting another. When no compatible GPU
capacity is available, the session waits and `antioch session status`
shows its place in the queue.

`session start` stays attached, waits for readiness, and keeps the session
alive while it waits.

Every command that needs a session accepts `--session SESSION`. Without it,
the CLI uses the session it last used in this worktree, then the sole live
interactive session. The CLI stops on ambiguity.

## Run a command in a service

```bash
antioch services exec python src/main.py
antioch services exec python src/main.py --seconds 60
```

`services exec` defaults to the simulator service, or the only
active service when there is no simulator. Use `--service` to select a
helper and repeated `--profile` options to enable authored profiles at startup.
Exec starts or reuses an interactive session and streams output
and exit status. A newly created session builds from the current project.
With a matching local project, each command applies the target service's
development watch rules once before execution; rebuild rules do not fire on
that pass. Run `antioch services watch` to apply changes continuously during
an edit loop. `--timeout` stops the command after
that many seconds (default 900). The command requests the session's Isaac GUI
stream by default; `--stream` states that request explicitly, and
`--no-stream` runs the process headless and leaves the one session stream to
another process. The flag sets `ANTIOCH_PROCESS_STREAM` to `1` or `0` in the
process environment, and `antioch.start_simulation()` reads it.

Exec preserves stdin and literal argv. It selects a PTY when local stdin and
stdout are terminals and forwards resize events; `--tty` and `--no-tty` override
that choice. A non-PTY command keeps stdout and stderr separate and receives
EOF when local stdin ends. The CLI returns when the command exits without
waiting for local input to close. Ctrl-C stops the exact remote process with
bounded escalation; it does not release the session.

## Inspect an existing service

```bash
antioch services exec --session SESSION --service sim -- nvidia-smi
antioch services logs --session SESSION SERVICE...
antioch shell --session SESSION
antioch services cp sim:/workspace/project/output.png ./output.png
```

With `--session`, these commands use that exact session. Without it, `exec`
can start a session; logs, shell, and copy require an existing one.

- `exec` runs a finite command and relays its output.
- `logs` reads service entrypoint output.
- `shell` opens an interactive PTY in one service.
- `cp` requires exactly one endpoint in `SERVICE:PATH` form.

Use the command help to select a supporting service or change timeout and
output behavior.

## Apply local development changes

```bash
antioch services watch
antioch services restart
```

`watch` follows the manifest rules continuously. The available actions are
`sync`, `sync+restart`, `sync+exec`, and `rebuild`. They update a live
interactive session only. A rebuild rule captures the declared build context,
skips an already verified build-key result, creates a new immutable project
revision when needed, and advances the service after the replacement is
healthy.

`restart` restarts selected service processes in the same session. It does
not create a new session.

## Use named routes

Declare the route in `antioch.yaml`, including its direction. Then bind it for
the selected session:

```bash
antioch services ports
antioch services ports --bind sim.viewer=127.0.0.1:8080
antioch services ports --clear sim.viewer
antioch services ports --serve
```

Use a bind for `client-to-service` routes. `--serve` forwards every mapped
route on its loopback address to the session until you stop the
command. Nothing forwards while `--serve` is not running.

## Use Jupyter

```bash
antioch jupyter lab
antioch jupyter cell 'print("ok")'
antioch jupyter cell --stream 'import antioch; antioch.start_simulation()'
antioch jupyter lab --stop
```

JupyterLab runs in the selected interactive session's remote simulator service
on its reserved `jupyter` route; add `--session SESSION` to name another one.
`jupyter cell` uses Jupyter REST and WebSocket APIs through that route, runs
on the one live kernel, and starts a kernel when none is live; JupyterLab's
own controls manage several. Jupyter never creates a session. Interrupt
`antioch jupyter lab` or run `antioch jupyter lab --stop` to stop JupyterLab.
Kernels start without the GUI stream. Pass `--stream` on the cell that calls
`antioch.start_simulation()` when the user wants to watch its Isaac GUI; the
declaration applies to that cell only, and `--no-stream` refuses the stream
for a cell beside a streamed process.

After `antioch.start_simulation()`, import a source-backed decorated scenario and make a
**programmatic call**:

```python
from src.scenarios import falling_cube

results = falling_cube(drop_height=4.5)
```

Inside an Antioch-managed session command, a programmatic call creates a saved
scenario run. Outside managed compute, it remains local and temporary. Use
`antioch scenario run` when the CLI must select and monitor the execution.

Use `antioch jupyter --help` for Lab, cell, and kernel commands on the
selected session.

## Stop the session

```bash
antioch session stop --session SESSION
```

Stopping a session stops the services and removes the temporary file system.

Once ready, an interactive session stops by itself after 15 minutes without a
heartbeat. An attached CLI command renews the heartbeat, so this happens only
when no command holds the session. The idle countdown does not run while the
session is still waiting for compute.

Use the webapp's
**Usage** page for organization GPU, CPU, and memory totals.
