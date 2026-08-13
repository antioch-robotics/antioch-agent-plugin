# Inspecting scenario runs: `antioch scenario`

Each scenario run keeps its outcome, exact inputs, logs, named results,
telemetry, and artifacts. This run history is shared with the organization.
Suite runs have their own history; see `references/suites.md`.

## Local discovery

- `antioch scenario collect --json` lists authored scenarios with their params,
  cases, and tags. Pass a concrete path, such as `antioch scenario collect
  scenarios --json`, to narrow discovery. This command is local and does not
  allocate a machine.

## Listing and filtering: `scenario list`

```
antioch scenario list --mine --since 7d
antioch scenario list --suite nightly --outcome failed
antioch scenario list --result 'final_z:<:0.4' --json
```

Use `antioch scenario list --help` for the complete filter set. Common filters
include scenario, project, suite, user, tag, parameter, result, time range,
dispatch type, phase, and outcome.

- `-q/--search` matches a scenario-name substring; `--scenario` is the exact
  authored name.
- Provenance filters: `--suite-run-id ID` (one invocation's members),
  `--invocation-id ID` (everything one queued submission created — queued
  `--json` output carries the id), `--dispatched-from HOST` (a hostname, or
  `'Mission Control'` for workspace dispatches), and `--no-suite`
  (standalone runs only).
- `--param` / `--result` filter on scenario params and reported metrics. The grammar is `key:op:value`, repeatable — every predicate must match:
  - `=` typed equality for scalars (`--param 'seed:=:42'`), containment on array/object values
  - `~` case-insensitive substring: `--param 'label:~:warehouse'`
  - `<` / `>` numeric ordering: `--result 'final_z:<:0.4'` (values must be numeric)
  - `@` containment along a dotted path: `--result 'grasp.quality:@:pass'`
  Dotted keys are **literal** for every other operator — only `@` traverses a dotted path, so `--result 'grasp.quality:=:pass'` matches a key literally named `grasp.quality`, not a nested field.
- `--since` / `--until` take an ISO-8601 datetime (read as UTC without an offset) or a duration ago: `30m`, `2h`, `7d`.
- `--dispatch interactive` selects foreground runs; `--dispatch queued`
  selects queue workers.
- `--phase` and `--outcome` are repeatable. `completed` includes every terminal
  outcome; use `--outcome` to narrow it.
- Everyday examples: `--mine --since 7d` (my recent runs), `--suite nightly --outcome failed` (last night's failures), `--dispatch interactive` (foreground runs only).
- `--json` list pages are `{ "items": [...], "next_cursor": "..." }`; pass
  `next_cursor` back as `--cursor` and keep fetching while it is non-null. A
  cursor pins its page's whole query — repeat no filters beside it.

Scenario list and show JSON use the same customer-facing schema. It includes
run and suite identities, project, machine and dispatch provenance, authored
inputs, lifecycle and outcome, timings, results, logs, artifacts, and
capability flags.

## Discovering filter values

- `antioch scenario suggest tag --json` returns distinct stored values and
  counts for one filterable field. The fields are `user_email`, `tag`,
  `suite`, `scenario`, `project`, and `dispatched_from`; a narrowing prefix
  is an optional second positional, as in
  `antioch scenario suggest scenario bin --json`. Reach for suggest before
  guessing any filter value.

## Inspecting one run

- `antioch scenario show SCENARIO_RUN_ID --json` — the customer-facing run:
  phase, outcome, parameters, results, artifact index, timing, and links to its
  suite, project, and command. Integer timestamps are Unix microseconds.
- `antioch scenario show SCENARIO_RUN_ID --logs --service sim` — show captured
  logs for one project service. Bare `--logs` lists every service with its size,
  then shows `sim` (the primary run diagnostic); pass `--service` for another
  stream. The webapp Output view exposes the same service selector.
- `antioch scenario logs SCENARIO_RUN_ID` — the run's captured process output, stdout and stderr kept separable under redirection. `--stdout` / `--stderr` isolate one stream (mutually exclusive); `--follow` starts from the newest stored output and polls until the run completes; `--json` emits the assembled entries. Logs are the first stop for a failed run; metrics are the first stop for a passed-but-suspicious one.
- `antioch scenario download SCENARIO_RUN_ID` — downloads artifacts directly
  from object storage, verifies their declared digests, and prints each destination.
  Without `-o`, the target behavior is a per-run subdirectory under the
  current directory, so one download cannot collide with another run. `-o
  PATH` selects an explicit destination. The download **includes the `.rrd`
  Rerun telemetry** when the run kept one. `--json` emits one transfer
  manifest with `scenario_run_id`, `destination`, `files`, and `count`; without
  it stdout prints one destination path per file. `--artifact NAME` picks
  artifacts (repeatable; omit for
  all) — NAME is the **logical artifact key** from the run's artifact
  index (`telemetry`, not `telemetry.rrd`); the JSON's artifact index maps
  each key to its stored filename, and a wrong key's error lists the valid
  keys. `--force` overwrites. The `.rrd` opens in Rerun.
- Files left only on a machine are temporary. If it is not a result, artifact,
  log, or telemetry, do not treat it as durable.

## Cancellation

- `antioch scenario cancel SCENARIO_RUN_ID --json` cancels one standalone
  queued or running scenario run; the JSON result carries `changed`, and an
  already-terminal run is reported as a note rather than an error. A suite
  member cannot be cancelled alone — cancel its parent with
  `antioch suite cancel SUITE_RUN_ID --json` (`suites.md`).

## Deletion

- `antioch scenario delete --run RUN_ID` deletes standalone completed runs;
  repeat `--run` for more than one. Machine usage already measured stays
  billable. Suite parents have their own home:
  `antioch suite delete --run SUITE_RUN_ID` deletes one suite run and its
  members, while `antioch suite delete --suite NAME` deletes every owned run
  of that authored suite. A suite member cannot be deleted apart from its
  parent. `--yes` skips confirmation, and `--json` requires `--yes`. Runs
  are shared with the organization, but only their owner can delete them.
  Confirm the scope with the user first.
