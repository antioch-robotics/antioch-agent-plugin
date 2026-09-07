# Work with scenario history

A scenario is a saved simulation test. Each run includes its selected
inputs, resolved service image digests, outcome, checks, named results, logs,
telemetry, and artifacts.

## Preview and submit

Collection imports scenario modules on the client and spends no remote
compute:

```bash
antioch scenario collect
```

Keep Isaac imports inside scenario functions so collection works without
Isaac installed.

Submit one scenario. Interactive submissions report progress until they
finish:

```bash
antioch scenario run --scenario falling_cube --set drop_height=4.5
```

Use `antioch scenario run --help` for case, tag, path, and output selection.
The command validates the selection before it submits a run.

By default, scenario execution uses the selected interactive session and stays
attached until the run finishes. Add `--detach` for unattended execution on
reusable background sessions. The scenario record stays independent of the
session and retains its result after that session stops.

An attached run requests the session's GUI stream by default and shares
`--stream/--no-stream` with `antioch suite run`; a detached run is headless.
Each scenario reserves the session livestream while its simulation process
runs; Mission Control can show that stream.

## Read the result

```bash
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario show SCENARIO_RUN_ID --logs
antioch scenario download SCENARIO_RUN_ID
```

Read the terminal state, checks, and error before you diagnose a failure. The
download command retrieves the saved output bundle, including recorded
viewer data when present.

Use the webapp to compare parameters, checks, numeric results, logs,
telemetry, and artifacts on one page.

## Find a run

```bash
antioch scenario list --json
antioch scenario suggest tag --json
```

Discover stored values before adding a filter. Use
`antioch scenario list --help` and `antioch scenario suggest --help` for the
current searchable fields.

## Cancel or repeat work

```bash
antioch scenario cancel SCENARIO_RUN_ID
antioch scenario rerun SCENARIO_RUN_ID
```

Cancellation records a terminal result and signals active work. Completed
evidence remains.

A rerun receives a new ID and uses the original immutable project revision and
exact inputs. It does not replace the original run, rebuild, resolve mutable
tags, or use current local files. This preserves submitted inputs, not
execution conditions: scheduling, capacity, simulator timing, and external
asset availability can change, so a rerun can have a different outcome or
duration.

Use `antioch scenario delete --run SCENARIO_RUN_ID` only when the user
explicitly wants to remove a scenario run from history.
