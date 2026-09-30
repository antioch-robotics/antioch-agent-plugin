# Sessions and direct execution

You can start sessions with `antioch session new`; Antioch also starts and
reuses sessions for submitted runs. Both count toward your personal quota across projects and the organization total until their stop is accepted. A stopping session no longer holds quota. Commands
can use either, but implicit dispatch never borrows a session you started.
`antioch session list` shows who holds each session.

## Select a session

| Selection | Session used |
|---|---|
| `--session SESSION_ID` | Exactly that session of yours |
| `--run RUN_ID` | The session that run is on or ran on, while the session remains live; a run that never reached a session is refused |
| Neither flag | Your only running session of this project that you started; none or several is an error |

Direct commands never allocate or switch compute. `session status` and
`session release` take the session ID positionally, with the same default. An explicit ID can name your session in another project.
A command that chooses the session for you prints the selected session and service; naming the session explicitly avoids that extra notice.

```bash
antioch session new
antioch session list --json
antioch session status SESSION_ID --json
antioch service exec -- python src/main.py
antioch service shell --run RUN_ID
antioch scenario run NAME --session SESSION_ID
```

`session new` builds when needed, then queues for capacity. The session appears in the list once its images are ready. It adds a session
without replacing another. At your quota it refuses and identifies sessions
to inspect. Sessions already stopping are excluded. `session status` reports
`wait_reason`, `queue_position`, and the revision ID. Capacity waits can outlast a tool call; keep
the original command running and inspect it rather than starting duplicates.

`session new --profile PROFILE` activates the services in that profile and builds their revision. `--revision REVISION` instead starts an existing revision by exact ID or unambiguous prefix. If you also pass profiles, they must match the saved revision. A revision built for an incompatible older Antioch release is refused; omit `--revision` to build a current one.

`session new --idle-timeout 2h` changes the idle window from the default fifteen minutes. It accepts seconds or durations of at least one minute and at most one year; `never` disables automatic idle release. Choose a longer or unlimited window only when the task calls for it, since the session keeps using compute. `session list --all-projects` includes your other projects; `--verbose` adds GPU and region details and `--limit` bounds the list. `session new --verbose` exposes full build output; a failed build retains a bounded log tail.

## Run and sync source

```bash
antioch service exec --no-stream -- python src/main.py --seconds 60
antioch service exec --service sim -- nvidia-smi
antioch service restart
antioch service ps --json
antioch service logs --service SERVICE
antioch service cp sim:/workspace/project/output.png ./output.png
```

Antioch options precede `--`; everything after it is the remote command.
It runs in `/workspace/project` and returns the process exit status. Pass `--stream` to watch the simulation in Mission Control. Without it, Isaac runs headless; `SimulationConfig(stream=False)` also keeps a process headless when the flag is present. Ctrl-C stops the process, not its session.

Exec, shell, Jupyter, and restart copy the manifest's `sync` mappings first;
attached commands keep copying local edits. A watcher that has not seen another local edit leaves a later run's installed bundle alone; the next edit or new command applies the checkout again. Local files win, removed mapped
files are deleted, and files created remotely are left alone. A partially written local file waits for another sync pass. `.gitignore`
and mapping exclusions apply. Changed files transfer whole, so exclude large
frequently changing outputs or copy them separately. Copy paths must be inside `/workspace/project`; use a remote command to move files elsewhere. A separately copied file is not owned by a source mapping, so sync leaves it alone. Read [manifest mappings](manifest.md#source-sync) when local files do not reach the expected remote path.

`--run` selects the run's session; it does not make a command read-only or disable sync. This applies to exec, shell, and Jupyter even after the named run completed, while its session remains live. Logs and process inspection read that session without copying files. Restart, file copy, and port commands use `--session` instead of `--run`.

Every submitted run carries its own source bundle and applies it at startup.
Command sync still overwrites files while a run executes, so the last writer
wins. A later run reapplies its own bundle. Images and manifests stay frozen:
start a new session for dependency or image changes.

A run's bundle reaches every mapped service, but only its declared `restart_services` restart. A helper whose mapped files now differ from the files its process started with keeps running earlier code. The run verdict and `scenario show` identify that helper. Add it to the scenario's restart policy or restart it explicitly after the active scenario ends; see [scenario execution policy](../../scenario-design/SKILL.md#inputs-and-execution-policy).

Restart syncs source, then the session's machine restarts each service process in its existing container in dependency order, waiting for its fresh health check before the next, and then restarts dependents whose `depends_on` entry sets `restart: true`. A service that exits or fails its check afterwards fails the restart by name; a run whose restart fails ends with that reason. Use it when a long-running service loaded a file at startup and must read the new version. An idle service with no command has nothing to restart. An executing scenario blocks restart; a timeout does not undo startup, so inspect state before retrying. Use repeated `--service` options to restart only the helpers that need it. Profiles are fixed when the session starts: they cannot be enabled by selecting a service later.

`service ps --json` reports `process_healthy` for supervisor process containment, not the service's application health check. `session_ready` is aggregate readiness including authored checks. Use `session status --json` for readiness conditions and `service logs --tail 100` for a bounded entrypoint log read. With one active service, bare `service cp SRC DST` uploads; a `SERVICE:PATH` endpoint makes the direction explicit.

## Named routes

Keep a port forwarder running for declared routes:

```bash
antioch service ports --bind sim.viewer=127.0.0.1:8080
antioch service ports --serve
```

Declare ports first in the [manifest](manifest.md#routes). For `client-to-service`, the binding is the local listener; for `service-to-client`, it is the client destination. Forward routes get loopback bindings by default; reverse routes need an explicit destination. `--clear sim.viewer` resets a forward binding or removes a reverse destination. Bindings persist for this worktree, but no traffic flows until `--serve` runs. A declared mapping alone does not occupy a session; an open connection does. Stopping the forwarder closes connections, not compute. Reverse routes keep TCP connections and UDP source endpoints separate, permit up to 32 peers per registration, and expire idle UDP associations after 60 seconds. For discovery and message transport across ROS services, read [ROS 2](ros2.md).

## JupyterLab

```bash
antioch jupyter lab
antioch jupyter cell "print('ok')"
antioch jupyter stop
```

Jupyter uses the same session selection and never allocates compute. `--service` selects among compatible services; each service has its own server and credentials. Stopping one leaves the others running. Cells
select the sole kernel or create one; use `--kernel KERNEL_ID` among several.
A kernel does not start Isaac automatically. Pass `--stream` on the cell that starts Isaac to watch it. Later cells reuse the same engine and cannot change its stream mode.
Interrupting Lab disconnects the client; `jupyter stop` ends Lab and its
kernels. Notebook files are remote; copy them home or record their outputs before releasing the session. For iterative code and images, read the [Jupyter MCP guide](../../agentic-simulation/references/jupyter.md);
for saved evaluations, read [in-session recording](../../scenario-design/references/recording.md).

`jupyter lab --no-open` prints its private local launch URL without opening a browser. Open that exact file URL on the same computer and keep the command running for its tunnel; do not copy the sign-in file or token to another device. Ctrl-C detaches the local tunnel without killing kernels.

`jupyter cell` defaults to a 900-second cell timeout and accepts `--timeout` for a different bound. JSON success goes to stdout; a failed cell exits 1 with `error.details.cell_result` on stderr when Jupyter returned a result. Ordered outputs can include images and display updates, are capped at 8 MiB, and mark omissions with `outputs_truncated`. A connection failure does not invent a cell result, and the CLI does not retry cells.

## Release

```bash
antioch session release
antioch session release SESSION_ID --force
```

Release tears down the selected session asynchronously. As soon as the stop is accepted, it no longer counts toward your personal or organization quota, and its usage ends at that time. Teardown can continue afterward. Active commands need
`--force`; a scenario is cancelled either way. Other suite members continue.
Save remote outputs first. Closing Python or an adapter does not release compute.

A session stays active while commands run, a scenario runs or is queued for it, a notebook cell executes, or a client such as JupyterLab or a port forward is connected. An idle default Bash prompt can expire when the runtime can confirm that no jobs are running. An idle Jupyter server or kernel alone, stream viewers, status reads, service entrypoints, and open console tabs do not keep the session active. A connected Jupyter server still counts as running work for `antioch session release`; run `antioch jupyter stop` first. Keeping a local adapter open does not reserve the session when it has no active work or forwarded client connections.

Sessions you start release after fifteen idle minutes by default; sessions Antioch starts release after five. `--idle-timeout` changes the window on a session you start, and `never` disables automatic release. `session status` reports what keeps the session active or when it will retire. If the service cannot report whether work is active, Antioch keeps it running rather than assuming it is safe to stop. Usage continues until the stop is accepted. Save remote files before releasing the session; upgrading the local SDK does not replace an older running service agent.

## Output, terminals, and streams

Exec and shell forward stdin and return when the remote process exits. Terminal detection enables a PTY; `--tty` and `--no-tty` override it. A PTY combines stdout and stderr and forwards terminal resize events. Use `--no-tty` for a piped command that must read until EOF: a PTY cannot half-close its write side, so local EOF sends no synthetic Ctrl-D. An interactive shell requires a local terminal. Output buffers are bounded; an output-gap notice means some bytes were no longer available, not that the remote program stopped.

Each Isaac service has one native video producer. A second process cannot claim an occupied stream; leave out `--stream` for commands that do not need it. Mission Control can show the selected service's stream while its simulator is running. A stream does not step physics or keep an idle session alive. [Simulation configuration](simulation-code.md#simulationconfig) controls startup and picture quality; [viewport capture](../../agentic-simulation/references/viewport.md) returns inspection images to the agent, and [telemetry](../../scenario-design/references/telemetry.md) saves run evidence.

For automation, the [Python client](python-client.md#use-an-existing-session) exposes the same session and service operations. Return to the [platform guide](../SKILL.md#capability-guides) for builds, run dispatch, and assets.
