# Define and run suites

A suite is a named, repeatable selection of scenarios and cases. Define it in
`antioch.yaml`:

```yaml
suites:
  acceptance:
    description: Warehouse acceptance checks
    select:
      - tags: ["warehouse", "smoke"]
        exclude_tags: ["slow"]
      - scenarios: ["dock_alignment"]
        cases: ["narrow", "wide"]
```

Fields inside one selector all apply. Separate selectors form an ordered
union. A selector can match paths, scenario names, case IDs, required tags,
and excluded tags.

## Preview before submission

```bash
antioch suite collect
```

Collection runs on the client. It shows the exact scenario and case expansion
without starting remote compute. Fix an empty or unexpected selection before
submission.

## Submit the suite

```bash
antioch suite run acceptance
```

Antioch records the suite's selected inputs and immutable project revision.
The suite groups child scenario runs in authored order. Interactive execution
is serial on the project's session and stays attached until the suite
finishes. `--detach` uses reusable background sessions automatically, within
your quota and available capacity. Closing the terminal after a detached submission does not cancel
the suite.

An attached suite requests the session's GUI stream by default, one child at a
time, and shares `--stream/--no-stream` with `antioch scenario run`; a
detached suite is headless. Mission Control can show the active stream.

Use `antioch suite run --help` for the current output and follow behavior.

## Read progress and results

```bash
antioch suite show SUITE_RUN_ID --json
antioch scenario logs SCENARIO_RUN_ID
antioch suite summary
```

The suite run links every selected scenario result. Read the failing
scenario checks and each member's output through `antioch scenario logs` on
its scenario run ID before you change the implementation. The
webapp can compare runs from one suite.

## Cancel or repeat a suite

```bash
antioch suite cancel SUITE_RUN_ID
antioch suite rerun SUITE_RUN_ID
```

Cancellation signals active work and prevents unstarted work from beginning.
Completed scenario evidence remains.

A rerun gets a new ID and uses the original immutable project revision and
parameters. Unbuilt interactive edits are not preserved. It does not change the
original suite, rebuild, or resolve
mutable tags again. Exact inputs do not guarantee the same outcome or timing
when scheduling, capacity, simulator timing, or external assets differ.

Use `antioch suite delete --run SUITE_RUN_ID` only when the user explicitly
wants to remove a suite run from history.
