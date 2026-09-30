# Suites

A suite is a named selection of scenarios and cases in `antioch.yaml`:

```yaml
suites:
  acceptance:
    description: Warehouse acceptance checks
    select:
      - tags: ["warehouse", "smoke"]
        exclude_tags: ["slow"]
      - scenarios: ["dock_alignment"]
        cases: ["narrow", "wide"]
      - scenarios: ["cube_bounce"]
        set: {drop_height: [1, 2], restitution: 0.5}
```

Fields in one selector all apply; separate selectors form an ordered union.
A selector takes the same fields as `scenario run`: `paths`, `scenarios`,
`tags` and `exclude_tags` choose scenarios; `cases`, `all_cases` and `rows`
choose rows; `set` overrides parameters, and a list under one key crosses
into a grid.

## Collect, run, and inspect

```bash
antioch suite collect --json
antioch suite run acceptance
antioch suite show SUITE_RUN_ID --json
antioch suite list --suite acceptance --json
antioch suite summary --json
```

Optionally label an invocation with `antioch suite run acceptance --run-name "Controller v2"`. In Python, pass `run_name="Controller v2"` to `client.suite.run(...)` or `client.suite.submit(...)`; see [the Python client](python-client.md). The name belongs to this run, not the suite definition. It is a single line of up to 120 characters, does not need to be unique, and is returned as `run_name` in the suite record. Continue using `suite_run_id` to show, follow, cancel, rerun, or delete a run. The console uses the name in run history and comparisons and shows the submission date for unnamed runs. Omitting the name preserves existing behavior, including older records. Reruns preserve the saved name and receive a new ID.

Collection expands local definitions without compute. A suite records its
selected inputs and a frozen revision, with one child scenario record per
member. Children run on Antioch-managed sessions, 2 at a time unless
`--parallel` says otherwise; with `--session` they queue on that session of
yours one at a time. Each child applies the suite's source bundle when it
starts, on any session, so later local edits do not reach queued members.
`suite run --detach` returns after submission; use `antioch suite show SUITE_RUN_ID --follow` to follow it later. Following,
`--session`, `--parallel` and time limits match [scenarios](scenarios.md).

Each child's `@antioch.scenario(service="fluoro")` selects its execution
container and reruns keep it; when several services are eligible, scenarios
without `service=` are ambiguous. Live Rerun follows the active child; native Isaac/WebRTC video follows the selected service when it provides a stream. Pass `--stream` to request live Isaac video and live Rerun telemetry for the suite; `SimulationConfig(stream=False)` still suppresses a scenario's video. Without the flag, runs are headless and telemetry is only the saved `telemetry` artifact.

A suite continues after a child fails, errors, or times out. Releasing an
Antioch-managed session ends that child as `session released`; its siblings
continue. Read each failing child's checks, logs, and artifacts by its
scenario run ID. Summary is a bounded view over recent runs; show reads one
invocation.

## Cancel and repeat

```bash
antioch suite cancel SUITE_RUN_ID
antioch suite rerun SUITE_RUN_ID
antioch suite rerun SUITE_RUN_ID --set seed=7 --timeout 600 --stream
```

Cancel stops active work and prevents unstarted members from starting;
completed evidence remains. Reruns use saved images and inputs under new IDs,
with or without `--session`. `--set`
reaches every member that has the parameter (a key no member has is refused,
values keep the saved type, a comma list is a grid of suite reruns);
`--timeout` and `--stream/--no-stream` replace the saved values.
`antioch suite delete --run SUITE_RUN_ID` previews and confirms deletion of a completed suite and its children, within the user's request.

Design child checks with [scenario design](../../scenario-design/SKILL.md), read each child through [scenario history](scenarios.md#find-and-analyze), and use [the Python client](python-client.md) for automated selection and comparison. [Antioch platform](../SKILL.md#capability-guides) links the remaining operations.
