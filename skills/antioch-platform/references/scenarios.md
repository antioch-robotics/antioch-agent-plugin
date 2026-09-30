# Scenario execution and history

A scenario definition is a Python function decorated with `@antioch.scenario`; [scenario design](../../scenario-design/SKILL.md) owns typed inputs, checks, and evidence. Each invocation has a run ID
and a record with author, timestamps, inputs, checks, results, and artifact
descriptors. Phase describes execution progress; outcome its verdict.

Managed runs pin service images and a source bundle of the mapped project
files at submission, captured with the same `sync` rules a command uses.
Each run applies its bundle when it starts, on any session, so later local
edits do not reach it; a command that syncs onto the session while it
executes changes its files, the last writer winning.
Scripts and notebooks inside a session can also record by calling a scenario
directly ([in-session recording](../../scenario-design/references/recording.md));
those records have no managed rerun. A submitted bundle updates service files without restarting every service; see [stale helper processes](sessions.md#run-and-sync-source) when a run reports earlier service code.

## Collect and run

```bash
antioch scenario collect --json
antioch scenario run falling_cube --set drop_height=4.5
```

Collection imports local source without compute or executing the body; it
does not prove native API calls or task logic work.

Submission uploads the source bundle and, without `--session`, builds the
current revision when needed and runs on Antioch-managed sessions within your
quota, reusing idle ones that serve the same images. Runs queue oldest first,
at most 2 at once per command (`--parallel 4` runs up to 4, still capped by
your session limit); Antioch never stops a session to make room, and one it
started ends five idle minutes after its last run. `--session SESSION_ID`
queues the runs on that session of yours, user- or Antioch-managed, pinned to
its images with nothing built, one at a time; `--run RUN_ID` names the session
a run is on or ran on instead, while that session remains live. The command follows the runs; `--detach` returns once the runs are submitted, and `--verbose` adds captured process output while following. Ctrl-C stops following without cancelling. Queued records distinguish a command's parallel limit, your session limit, and available capacity; inspect the wait reason before resubmitting. Resubmitting under the same `--invocation-id` returns the existing
records, so a submission retried after a lost connection is not doubled.

Names, `--paths`, `--tags` and `--exclude-tags` choose scenarios, with `*`
and `?` globbing (`'*grasp*'`); `--cases`, `--all-cases` and `--rows @FILE`
choose rows. `--set KEY=VALUE` overrides a parameter on every row, and several
values for one key (`--set KEY=V1,V2`, or a repeated key) cross into a grid.
`scenario collect` with the same options previews the exact runs. Several
runs form one batch sharing an invocation ID, never a suite.

```bash
antioch scenario list --invocation-id INVOCATION_ID --follow
antioch scenario cancel --invocation-id INVOCATION_ID
```

## Set time limits

```bash
antioch scenario run falling_cube --queue-timeout 30m --timeout 900
```

`--queue-timeout` bounds admission and capacity waiting until a session starts launching for the run. It starts at admission after the client prepares and uploads the submission; any admitted build still in progress can count toward this wait. It no longer applies once a session starts launching, or while waiting behind another run on an existing session; it is not a deadline for the scenario body. A run whose waiting deadline passes fails with `queue timeout`. The default
is one hour and the maximum 24 hours; a rerun takes no `--queue-timeout` and
gets the default. `--timeout` bounds each scenario's execution, overriding the scenario's configured value or 900 seconds. Both accept seconds or duration strings such as `30m` or `2h`, and both apply to suite runs.

## Find and analyze

```bash
antioch scenario list --json
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario logs SCENARIO_RUN_ID --json
antioch scenario download SCENARIO_RUN_ID --output downloads --json
antioch scenario download --invocation-id INVOCATION_ID --output downloads
```

A download keeps files that already hold an artifact's exact bytes, so a
repeated download needs no `--force`; `--invocation-id` puts each run of a
batch in its own folder.

List filters cover scenario, suite, tags, parameters, results, phase/outcome, author, and time. Phases are `prepared`, `assigned`, `running`, `finishing`, and `completed`; outcomes are `passed`, `failed`, `skipped`, `cancelled`, `errored`, and `timeout`. `--search` matches a scenario-name substring, `--no-suite` excludes suite members, and `--mine` narrows to your runs; `--user` selects a member, `--suite-run-id` selects one suite invocation, and `--all-projects` widens past the current project. `--since 7d` selects recent submissions. Parameter and result predicates use `key:op:value`, with `=`, `~`, `>`, or `<` as the operator:

```bash
antioch scenario list --scenario dock_alignment --outcome failed --since 7d --json
antioch scenario list --param 'speed:>:0.3' --result 'max_error_m:>:0.1' --json
```

For nested results, prefix a dotted key with `@` to traverse it; prefix with `=` to read a literal key. `--view summary` on a list omits parameters, results, artifacts, and startup logs, so use `show` for the complete record. Lists return one page; pass `next_cursor` unchanged through `--cursor`, preserving the filters. Normal output defaults to 10 rows; JSON defaults to 50, and `--limit` accepts up to 200.

Finite JSON goes to stdout, progress and errors to stderr. `scenario list`
and `suite list` print one object, `{"items": [...], "next_cursor": ...}`; a
run's checks are `results["checks"]`, a list of `criterion`, `passed` and
`detail`. Followed JSON is line-delimited, one object per state change,
ending with a `"type": "summary"` line; structured errors include
`retryable`.

## Cancel, rerun, delete

```bash
antioch scenario cancel SCENARIO_RUN_ID
antioch scenario rerun SCENARIO_RUN_ID
antioch scenario rerun SCENARIO_RUN_ID --set drop_height=1,2 --timeout 600
```

Cancel signals active work and preserves completed evidence. A rerun creates
new records from saved images and inputs and runs the saved bundle, not local
edits. Antioch chooses a session unless you pass `--session SESSION_ID` or `--run RUN_ID` to select an existing live session, which must serve the saved revision. It needs no checkout. For grid reruns, `--parallel` controls concurrency only when Antioch chooses sessions; combining it with either session selector is refused. `--set`, `--timeout` and `--stream/--no-stream`
change only those inputs; `--set` spells a grid as on `scenario run`, a key
must be a saved parameter, and its value keeps the saved type. Bounds and
allowed values refuse before submission when the checkout is exactly the
pinned code, else when the rerun starts. In-session and
source-free records have no managed rerun.
`antioch scenario delete --run SCENARIO_RUN_ID` previews and confirms deletion of a completed standalone run. A suite child is deleted with its suite. Deletion is separate from cancellation and must be within the user's request. Deleting another member's records also requires `--all-owners` and sufficient authority.

Use [the Python client](python-client.md#collect-inspect-and-submit) to collect and submit these same selections programmatically. For grouped evaluations, continue with [suites](suites.md); for evidence that renders incorrectly, use [telemetry](../../scenario-design/references/telemetry.md). Return to [Antioch platform](../SKILL.md#capability-guides) for other operations.
