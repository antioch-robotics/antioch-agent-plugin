# Automate Antioch from Python

Use `antioch.Client` for the same operations the CLI performs. It returns typed records and handles instead of printing terminal output. Constructing it does not contact the platform; collection works locally, and credentials resolve on the first remote request. `Client(project="/path/to/project")` selects a checkout. With no path, it discovers the current project; outside a project, it can still read shared history.

This is the automation reference for [Antioch platform](../SKILL.md). Use [scenario design](../../scenario-design/SKILL.md) for code that runs *inside* a scenario. A client dispatches that code; it does not turn local Python into remote execution.

## Collect, inspect, and submit

```python
import antioch

with antioch.Client() as client:
    collected = client.scenario.collect("dock_alignment", set={"speed": [0.2, 0.4]})
    for spec in collected:
        print(spec.scenario, spec.case, spec.params)
    submission = client.scenario.submit(collected.runs, parallel=2)
    print(submission.invocation_id)
    outcome = submission.wait()
    for run in outcome.runs:
        print(run.scenario_run_id, run.outcome, run.results)
```

Collection imports definitions without authentication, a simulator, or compute. The returned `RunSpec` objects include the resolved case, parameters, tags, service, startup defaults, and source digest. Filter or reorder collected specs before submission; do not invent specs. Submission checks them against the current source, so recollect after editing the defining files.

`client.scenario.run(...)` combines collection and submission. Names or decorated callables select scenarios; `paths`, `tags`, `exclude_tags`, `cases`, `all_cases`, `rows`, and `set` match the [CLI selection rules](scenarios.md#collect-and-run). In Python, a sequence of `set` values creates a grid; a string stays one value even if it contains commas. Use `select=[antioch.Selection(...)]` for an ordered union of complete selections instead of mixing it with selection keywords.

`client.suite.run("acceptance")` collects and submits a named YAML suite. To inspect its members first, unpack the collection and submit its run specs:

```python
with antioch.Client() as client:
    (acceptance,) = client.suite.collect("acceptance")
    submission = client.suite.submit(acceptance.runs, name="acceptance")
```

Both suite methods accept an optional `run_name`, for example `client.suite.run("acceptance", run_name="Controller v2")`. On `submit`, `name` identifies the suite and `run_name` labels this invocation. Names are display labels, not identifiers; retain `submission.suite_run_id` for later operations. [Suite run names](suites.md#collect-run-and-inspect) covers limits and how unnamed runs appear in the console.

To run only part of a suite, select those scenarios directly. See [suites](suites.md) for membership, ordering, and child outcomes.

Both submission paths accept `session`, `run`, `parallel`, `timeout`, `queue_timeout`, `stream`, and `invocation_id`. Pass `stream=True` on a submission or `client.service.exec(...)` to request video. `SimulationConfig(stream=False)` can keep the process headless, but `SimulationConfig(stream=True)` cannot turn video on by itself; see [simulation configuration](simulation-code.md#simulationconfig). Session selection and frozen source follow [the session contract](sessions.md). Use `session` or `run` to target one existing session. Passing `parallel` with either is refused; runs on a selected session execute one at a time. Use `parallel` only when Antioch chooses sessions. Closing the client releases its connections, not sessions or submitted runs.

## Follow or recover a submission

A `Submission` exposes `invocation_id`, `scenario_run_ids`, and, for a suite, `suite_run_id`. `acknowledged` reads the records already returned at admission without another request; `runs` fetches their current state. `events()` yields changes until completion, and `wait()` returns terminal runs and their aggregate verdict. Stopping either wait detaches without cancelling; `cancel()` explicitly stops unfinished work.

Persist the invocation ID before relying on a long-lived connection. Another process can call `client.submission(invocation_id)` to attach again. `SubmissionUnknown` means the admission reply was lost and work may already exist. Read `error.invocation_id`, look it up, and retain the same ID if retrying the same submission. Do not issue a fresh ID as an automatic response to a lost reply. Reusing an ID with different inputs is a conflict.

A successful aggregate allows passed or skipped runs; inspect each run's outcome and checks when skips matter to the task. Submission or process success alone does not establish that the simulation met its criteria.

## Use an existing session

```python
import sys

import antioch

with antioch.Client() as client:
    with client.service.exec("python", "src/main.py", session="ses-...", tty=False, stream=False) as process:
        for frame in process.frames():
            output = sys.stderr.buffer if frame.stream == "stderr" else sys.stdout.buffer
            output.write(frame.data)
            output.flush()
        exit_code = process.wait()
    print(exit_code)
```

This example needs an existing session. `client.session.new()` builds and starts one; `idle_timeout="2h"` sets a custom idle window and `idle_timeout="never"` disables idle release; `list()`, `status(session_id)`, `watch(session_id)`, and `release(session_id)` inspect and manage it. A direct command never allocates compute. `session=` names a session and `run=` resolves the still-live session a run used where the method supports it. Omitting both uses your only running session of the project that you started.

`Process.frames()` yields output; `wait()` returns its exit status without printing output. The example disables the PTY to keep stdout and stderr separate. `signal()` and `terminate()` explicitly stop work. `close()` ends source sync and the local connection; it is not a release operation. `client.service.ps()` inspects services and commands, `logs()` reads service entrypoint output, `restart()` reloads their configured commands after sync, and `cp()` moves files under `/workspace/project`. `ports()` returns route bindings; its `serve()` method keeps forwarding until stopped. See [sessions](sessions.md) for sync, restart, routing, streaming, and idle behavior.

`client.jupyter.cell(code, session=..., kernel=...)` syncs source and starts JupyterLab or a kernel if needed. It returns a cell outcome and preserves kernel state. Check `result.status == "ok"` or `result.exit_code == 0`: a Python error in the cell returns a failed result, not an exception. Connection and input failures can still raise. A running cell or an open connection through the forwarded Jupyter port keeps the session active; an idle server or kernel alone does not. Retaining a client or kernel handle is not a reservation. Kernel state is lost when its session ends; `client.jupyter.stop(...)` stops its server and kernels immediately. Use the [Jupyter guide](../../agentic-simulation/references/jupyter.md) for startup, module reloads, capture, and recovery.

## Inspect history and reusable assets

`client.scenario.list(...)` and `client.suite.list(...)` return one page with `items` and `next_cursor`; pass that cursor unchanged for the next page. `show()` reads one record, `watch()` follows changes, and scenario `logs()` and `download()` retrieve evidence. Suite `show()` returns the parent and its members; `summary()` groups recent suite history. Use [history filters](scenarios.md#find-and-analyze) to select the relevant records before paging through a large project.

Scenario and suite `rerun()` return new submissions using saved images, inputs, and source, with supported parameter, timeout, stream, and session overrides. They do not run local edits. `delete(runs=[...])` previews a deletion and returns a confirmation token; executing it requires the same selection and token, within the user's authorized request. Completed suite children are deleted with their suite, not individually.

`client.asset.list/show/pull/push/verify/repair` manage the [asset catalog](assets.md). The top-level `antioch.fetch_asset`, `load_asset`, and `save_asset` helpers serve simulation code. Private image credentials use `client.registry.list/login/logout`; [authentication](auth.md) explains their project scope. Do not print secrets or change credentials to repair an unrelated lookup.

## Handle failures and notices

`Client(on_notice=...)` receives notices such as an implicitly chosen session, wait reasons, and deprecations; the client is silent by default. Check `AntiochError.error_type`, `http_status`, `retryable`, and `details` rather than parsing prose. Public subclasses include `Unauthenticated`, `NotFound`, `Conflict`, `QuotaExceeded`, `IncompatiblePlatform`, `SessionUnavailable`, and `SubmissionUnknown`. Invalid local inputs raise `ValidationError`, which is not an `AntiochError`. Catch `antioch.SimulationError` when one handler should cover both; inspect the specific type when deciding how to recover.

A retryable failure says an unchanged request may succeed, not that the first request had no effect. Check operation identity before retrying mutations. Use [troubleshooting](troubleshooting.md) to separate input, access, capacity, startup, execution, and publication failures.
