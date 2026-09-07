# Choose and diagnose execution

Choose the workflow from the evidence the task must produce.

## Use `services exec` for one process

```bash
antioch services exec python src/main.py
```

Use `antioch services exec` when stdout, stderr, and the process exit status are the
complete result of a Python file. It starts or reuses a compatible interactive
session and streams the process output. A newly created session builds from
the current project. Use `services exec` for arbitrary commands or a named
supporting service. For later edits, run `antioch services watch` separately so the
manifest's watch rules update the live services. The command requests the
session's Isaac GUI stream by default; `--stream` states that request
explicitly. A session has one stream, so add `--no-stream` to run the process
headless beside a streamed scenario or kernel.

After `antioch.start_simulation()`, the SDK owns the process exit status. An
uncaught exception exits 1 with its traceback. `KeyboardInterrupt` exits 130.
A clean end exits 0. `sys.exit(n)` exits `n`. A bare `raise SystemExit(n)` at
the top level of a headless script exits 0 through Isaac's fast shutdown; use
`sys.exit(n)`. A script that replaces `sys.excepthook` without chaining the
previous hook takes that exit ownership with it.

The interactive session stays useful between commands. Stop it only when the
edit loop is done:

```bash
antioch session stop --session SESSION
```

## Use a scenario for saved evidence

```bash
antioch scenario collect
antioch scenario run --scenario falling_cube
```

Use a scenario when the result needs checks, named results, logs, telemetry,
artifacts, or later comparison. Collection runs locally. Dispatch records the
selected inputs and resolved service image digests.

The default command uses the selected interactive session and stays attached
until the run finishes. Add `--detach` to submit work that continues after the
terminal closes. Antioch uses reusable background sessions for that work.
Watch rules are an interactive development feature and do not alter the saved
scenario inputs.

Attached `antioch scenario run` and `antioch suite run` request the session's
GUI stream by default and share `--stream/--no-stream` with
`antioch services exec`; a detached run is headless. Each scenario reserves the
session livestream while its simulation process runs. The attached command
shows progress; Mission Control shows the live simulation.

## Use a suite for a named evaluation

```bash
antioch suite collect
antioch suite run acceptance
```

A suite expands the selectors in `antioch.yaml` and groups the resulting child
scenario runs in authored order. Interactive suites are serial on the selected
session. `--detach --parallel N` can use up to `N` reusable background
sessions. Closing the terminal after a detached submission does not stop the
suite.

## Diagnose from saved state

Read the saved result before changing code or submitting another run:

```bash
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario show SCENARIO_RUN_ID --logs
antioch suite show SUITE_RUN_ID --json
antioch suite show SUITE_RUN_ID --logs
```

The saved result tells you whether the failure came from admission, service
startup, the scenario process, a check, cancellation, or artifact handling.
Logs are grouped by service.

For an interactive failure, inspect the live session and then reach the
affected service:

```bash
antioch session status --session SESSION --json
antioch services logs --session SESSION SERVICE...
antioch services exec --session SESSION --service sim -- nvidia-smi
```

Confirm identity and version last:

```bash
antioch auth whoami
antioch --version
```

Do not infer a result from successful submission. Read the terminal run state
or follow it to completion.

## Repeat exact recorded inputs

```bash
antioch scenario rerun SCENARIO_RUN_ID
antioch suite rerun SUITE_RUN_ID
```

A rerun has a new ID. It uses the saved immutable project revision and exact
inputs. It does not rebuild, resolve mutable tags again, use later local edits,
or use a development watch transfer. Exact inputs do not guarantee the same
outcome or timing: scheduling, capacity, simulator timing, and external asset
availability can differ. Check catalog assets before rerunning and compare the
new evidence with the original.
