# Choose and diagnose execution

Choose the workflow from the evidence the task must produce.

## Use `service exec` for one process

```bash
antioch service exec python src/main.py
```

Use `antioch service exec` when stdout, stderr, and the process exit status are the
complete result of a Python file. It starts the project's session when none is
live, uses the live one otherwise, and streams the process output. A newly created session builds from
the current project. Use `service exec` for arbitrary commands or a named
supporting service. For later edits, run `antioch service watch` separately so the
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

The interactive session stays useful between commands. Release it only when the
edit loop is done:

```bash
antioch session release
```

## Use a scenario for saved evidence

```bash
antioch scenario collect
antioch scenario run --scenario falling_cube
```

Use a scenario when the result needs checks, named results, logs, telemetry,
artifacts, or later comparison. Collection runs locally. Dispatch records the
selected inputs and resolved service image digests.

The default command uses the project's interactive session and follows until
the run finishes. `--no-follow` returns after admission without changing that
session or stream intent. `--detach` selects headless background compute, and
`--follow` can wait for either mode. Antioch uses reusable background sessions for that work.
A background submission builds the current YAML independently of the project's
interactive session. Watch rules never run on background sessions. Source must
be included by the Dockerfile; run submission sends no source bundle.

Attached `antioch scenario run` and `antioch suite run` request the session's
GUI stream by default and share `--stream/--no-stream` with
`antioch service exec`; a detached run is headless. Each scenario reserves the
session livestream while its simulation process runs. The attached command
shows progress; Mission Control shows the live simulation.

## Use a suite for a named evaluation

```bash
antioch suite collect
antioch suite run acceptance
```

A suite expands the selectors in `antioch.yaml` and groups the resulting child
scenario runs in authored order. Interactive suites are serial on the project's
session. `--detach` distributes work across reusable background sessions
automatically, within your quota and available capacity. Closing the terminal after a detached submission does not stop the
suite.

## Diagnose from saved state

Read the saved result before changing code or submitting another run:

```bash
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario logs SCENARIO_RUN_ID
antioch suite show SUITE_RUN_ID --json
```

The saved result tells you whether the failure came from admission, service
startup, the scenario process, a check, cancellation, or artifact handling.
`scenario logs` prints a finished run's saved output, shows a running run's
current tail, and can follow it with `--follow`; read each suite member's
output through its own scenario run ID.

For an interactive failure, inspect the live session and then reach the
affected service:

```bash
antioch session status --json
antioch service logs SERVICE...
antioch service exec --service sim -- nvidia-smi
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

A rerun has a new ID. It uses the saved immutable project revision and
parameters. Unbuilt interactive edits are not preserved. It does not rebuild,
resolve mutable tags again, use later local edits,
or use a development watch transfer. Exact inputs do not guarantee the same
outcome or timing: scheduling, capacity, simulator timing, and external asset
availability can differ. Check catalog assets before rerunning and compare the
new evidence with the original.
