# Work with sessions

A session runs one project's services on managed compute. It is
`interactive` or `background`. Interactive sessions support `service exec`,
direct service access, development watch actions, routes, attached scenarios
and suites, and Jupyter. Background sessions are reusable compute pinned to a
project revision for unattended scenario and suite work.

Only interactive sessions take direct work. `session list` shows both kinds.

## Start the project's session

```bash
antioch session new
antioch session list
antioch session status
```

A project has one live interactive session. `session new` starts a fresh one
from the current manifest and images and replaces the project's current
session. On a terminal it confirms first because the replaced session's
temporary files are removed; without a terminal there is no prompt. Rome
refuses the replacement while the old session still runs a command unless you
pass `--force`, which skips the confirmation and stops that work first. A
build or quota failure leaves the current session unchanged.
Use profiles when optional supporting services are needed; inspect
`antioch session new --help` for the current syntax.

The default user quota is two interactive sessions across your projects and a
separate two background sessions. Release a session you do not need when the
quota is full. When no compatible GPU capacity is available, the session waits
and `antioch session status` shows its place in the queue.

`session new` stays attached, waits for readiness, and copies project files
from the sync rules before it returns. It keeps the session alive while it
waits. This initial copy never runs restart or exec actions; those remain
pending until `antioch service watch` runs.

Every command acts on the project of the current directory and its one live
interactive session; there is no session selector. Run the command inside the
project. Outside a project, a command that needs a session refuses.

## Run a command in a service

```bash
antioch service exec python src/main.py
antioch service exec python src/main.py --seconds 60
```

`service exec` defaults to the simulator service, or the only
active service when there is no simulator. Use `--service` to select a
helper and repeated `--profile` options to enable authored profiles at startup.
Exec starts or reuses an interactive session and streams output
and exit status. A newly created session builds from the current project.
Source enters a session at start and through `antioch service sync` or
`antioch service watch`. Every command runs against the source already in the
session; `service exec` never copies files. Run `antioch service watch`
during an edit loop. The command requests the session's Isaac GUI
stream by default; `--no-stream` runs the process headless and leaves the one
session stream to another process. `antioch.start_simulation()` reads that
request from `ANTIOCH_PROCESS_STREAM`. The process deadline and the terminal
allocation are options too: see `antioch service exec --help`.

Exec preserves stdin and literal argv. It selects a PTY when local stdin and
stdout are terminals and forwards resize events. A non-PTY command keeps stdout and stderr separate and receives
EOF when local stdin ends. The CLI returns when the command exits without
waiting for local input to close. Ctrl-C stops the exact remote process with
bounded escalation; it does not release the session.

## Inspect an existing service

```bash
antioch service exec --service sim -- nvidia-smi
antioch service logs SERVICE...
antioch service shell
antioch service cp sim:/workspace/project/output.png ./output.png
antioch service cp ./config.json sim:/workspace/project/config.json
```

These commands use the project's live session. `exec` can start a session;
logs, shell, and copy require an existing one.

- `exec` runs a finite command and relays its output.
- `logs` reads service entrypoint output.
- `shell` opens an interactive PTY in one service.
- `cp` copies in either direction. Use `SERVICE:PATH` for the remote endpoint.

Uploads and downloads accept only paths inside `/workspace/project` in the
service. Other paths, including `/tmp`, are refused. To download a file from
elsewhere, first copy it into the project with `service exec`. For example,
on a Python service:

```bash
antioch service exec --service sim --no-stream -- python -c "from shutil import copyfile; copyfile('/tmp/output.png', '/workspace/project/output.png')"
antioch service cp sim:/workspace/project/output.png ./output.png
```

For an upload to another service path, upload into `/workspace/project` first,
then use `service exec` to copy it to the final path. The exec command uses the
service's normal file permissions.

Use the command help to select a supporting service or change timeout and
output behavior.

## Apply local development changes

```bash
antioch service sync
antioch service watch
antioch service restart
```

`antioch service sync` copies matching files once. It never runs the restart
or exec part of a watch rule. `watch` applies `sync`, `sync+restart`, and
`sync+exec` continuously in the project's interactive session.

`restart` restarts selected service processes in their existing containers.
It waits for fresh authored health checks. A service with no effective command
stays idle, and restarting it does nothing. A reserved run can block restart.
The wait is bounded. A control timeout does not undo the process launch.
Check `session status` before issuing another restart: a new restart request
starts the process again and resets its startup clock. Watch retries retain
the same operation ID and continue waiting on the same process generation.

If health checks fail after initial readiness but the interactive session is
still running, use logs, sync, copy, or restart to inspect and repair it through
its existing agent. This does not make the session healthy or bypass an active
run's restart protection. A stopped, failed, or revoked session cannot be repaired
in place; start a fresh session instead.

The session's manifest, revision, images, and containers never change. For an
image or dependency edit, update the Dockerfile and run `antioch session new`.
The new session uses the new images and replaces the old one.

## Use named routes

Declare the route in `antioch.yaml`, including its direction. Then bind it for
the project's session:

```bash
antioch service ports
antioch service ports --bind sim.viewer=127.0.0.1:8080
antioch service ports --clear sim.viewer
antioch service ports --serve
```

For a `client-to-service` route, `--bind` sets the local listening address.
For a `service-to-client` route, it sets the destination on your client and
is required. For example, declare `sim.callbacks` on service port 9000, start
your local callback server on port 9100, then run:

```bash
antioch service ports --bind sim.callbacks=127.0.0.1:9100 --serve
```

The service connects to `127.0.0.1:9000`; the bridge sends its TCP bytes or UDP
datagrams to your local port 9100. Nothing forwards without `--serve`.
Stopping that command closes its listeners and connections, not the session.
Reverse routes need a current CLI and a fresh session with a supporting agent;
upgrading the CLI does not replace an older session's agent.

## Use Jupyter

```bash
antioch jupyter lab
antioch jupyter cell 'print("ok")'
antioch jupyter cell --stream 'import antioch; antioch.start_simulation()'
antioch jupyter lab --stop
```

JupyterLab runs in the project's interactive session's remote simulator service
on its reserved `jupyter` route.
`jupyter cell` uses Jupyter REST and WebSocket APIs through that route, runs
on the one live kernel, and starts a kernel when none is live; use
`--kernel KERNEL_ID` to select among several. Jupyter never creates a session. Interrupt
`antioch jupyter lab` or run `antioch jupyter lab --stop` to stop JupyterLab.
Starting a kernel does not start Isaac. A cell that calls
`antioch.start_simulation()` inherits Lab's preferred stream request. Pass
`--stream` to request the GUI explicitly or `--no-stream` for headless startup.
The choice travels with the cell and applies only during its execution; it
does not reconfigure a simulator that is already running. Older kernels still
accept ordinary cells but refuse these explicit flags before execution. Start
a new session, or set `SimulationConfig(stream=True)` or
`SimulationConfig(stream=False)` in the code that starts Isaac.

After `antioch.start_simulation()`, import a source-backed decorated scenario and make a
**programmatic call**:

```python
from src.scenarios import falling_cube

results = falling_cube(drop_height=4.5)
```

Inside an Antioch-managed session command, a programmatic call creates a saved
scenario run against the session's revision. Ordinary Python outside a session
also records by default, using a valid project and existing user or
personal-access-token credentials. A `with antioch.Scenario(...)` context can
record without a source file, including in a notebook cell; that saved run
cannot be rerun from history. Recording does not allocate a session or start
Isaac. See [caller recording](authoring.md#record-from-ordinary-python) for its
deadline and explicit offline mode. Use `antioch scenario run` when the CLI
must select and monitor the execution.

Use `antioch jupyter --help` for Lab, cell, and kernel commands on the
project's session.

## Release the session

```bash
antioch session release
```

Releasing a session stops the services and removes the temporary file system.
Background sessions belong to detached runs and are not released here; cancel
the run with `antioch scenario cancel` or `antioch suite cancel`.

Initial readiness starts the interactive session's 15-minute owner-client idle
lease. An attached CLI command renews access while it is connected. Listing or
checking session status does not keep a session alive; remote CPU activity is
not a keepalive. A later health failure or recovery does not reset the initial
lease. The idle countdown does not run while the session waits for compute.

Use the webapp's
**Usage** page for organization session time and recorded session attempts.
