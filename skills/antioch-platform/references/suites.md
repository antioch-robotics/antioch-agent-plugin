# Suites: named scenario selections

A suite is an ordered union of selector clauses saved in `antioch.yaml`. Use a
small suite for smoke checks and a broader one for acceptance; the definition
is the selection, so a suite run does not take ad-hoc scenario filters.

```yaml
suites:
  smoke:
    description: "Fast health checks"
    select:
      - paths: ["scenarios"]
        tags: ["smoke"]
  acceptance:
    description: "Warehouse acceptance matrix"
    select:
      - paths: ["scenarios"]
        tags: ["warehouse"]
      - scenarios: ["pick_and_place"]
        cases: ["nominal", "tight_clearance"]
```

Fields inside one clause narrow together; clauses are unioned in authored
order. Paths, scenario names, case ids, required tags, and excluded tags are
exact selection inputs (`manifest.md` owns the clause schema). Validate the
expanded catalog locally:

```bash
antioch suite collect --json
```

## Foreground suites

```bash
antioch suite run acceptance
```

Foreground execution uses the project's assigned machine(s), streams by
default, and returns a process verdict only after every selected scenario
finishes. Use the machine, timeout, stream, and output options printed by
`antioch suite run --help`; do not narrow the saved suite with scenario flags.
To run a subset, use `antioch scenario run` with its selection flags.

## Queued suites and reruns

```bash
antioch suite run acceptance --queue --json
antioch suite show SUITE_RUN_ID --follow --json
```

Queueing is the immutable-environment boundary. The submitter builds the
selected project services **on the project's current machine** — use
`machine checkout` to pick the stager — adds the current project source to
the simulation image,
pulls private images with the local Docker credential, and pushes the resolved
images into your organization's private registry. Antioch distributes the
suite's scenario runs
across eligible machines that run independently of the staging machine.
Queued workers are headless; development `watch`
rules and `ports` tunnels are not part of the queued environment. Do not
combine `--queue` with a typed `--stream`, `--verbose`, `--raw-logs`,
`--machine`, or `--machines`; `--json` on `suite run` requires `--queue`
(`running.md` explains the constraints).

Queued runs save their exact digest-pinned environment before Antioch
distributes them. For a single-machine interactive suite run, Antioch captures
the admitted `sim` container's project workspace at dispatch and then attempts
to save the exact images and source the suite run used. When that capture and
publish succeeds, queue the completed suite again exactly as it ran:

```bash
antioch suite rerun SUITE_RUN_ID
```

The webapp's Re-run button is the GUI twin of `suite rerun` and
`scenario rerun`. Multi-machine interactive runs are not currently
rerunnable. Repeat the
original command or use `--queue` when the result must be rerunnable. Antioch
explains when an older run or a failed source capture or publish leaves the
environment unavailable.

Standalone scenario selections use the same queue boundary:

```bash
antioch scenario run --tag smoke --queue --json
```

That form creates standalone scenario runs rather than a named suite
parent.

## History and cancellation

Use the read surfaces when a run id is not at hand:

```bash
antioch suite list --json
antioch suite summary --json
```

Inside a project both default to that project; add `--all-projects` to widen
the view. `suite summary` lists named suites with their latest-run state —
the fastest answer to "how is the nightly doing". Use the `--cursor` returned
by a JSON page to continue that same query. Cancel
queued or running work by its suite-run id:

```bash
antioch suite cancel SUITE_RUN_ID --json
```

Suite list, summary, and show JSON use a customer-facing projection. It keeps
suite and scenario identities, project and dispatch provenance, lifecycle and
outcome, child counters, timings, results, logs, artifacts, and rerun
capabilities. It omits tenant keys, raw user subjects, queue claims,
assignment and process identities, attempt counts, and generation fields. The
member records inside `suite show` use the same scenario projection as
`scenario list` and `scenario show`.

Finished member scenarios remain in the suite history; unclaimed queued runs are
cancelled and active processes are signalled. Check `antioch suite cancel
--help` before automating cancellation and confirm the organization-wide impact.

## Comparing suite runs

The webapp's Compare view takes two to eight suite runs and shows pass rate,
duration, and a scenario-by-scenario result matrix. The CLI has no `compare`
command — the agent path is JSON: pull each run with
`antioch suite show SUITE_RUN_ID --json` (its `scenario_runs` array carries
every member's outcome and results) and diff the fields that matter.

Suite `--phase` and `--outcome` filters are repeatable unions in both `list`
and `summary`; every finite JSON result is one document and every `*_at` value
is Unix microseconds. Followed JSON is explicit NDJSON. See `cli.md` for the
shared contract and each command's `--help` for its current fields.
