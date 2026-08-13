# Running code and scenarios

Antioch has two execution surfaces. `antioch run` is a one-off probe whose
stdout, stderr, and exit status belong to the Python process. `antioch scenario
run` selects authored scenarios and records their outcome, parameters, logs,
telemetry, and artifacts. Both need a `services.sim` entry in the manifest —
a service-only stack refuses Isaac dispatch until one is declared
(`manifest.md`).

## Start an interactive stack

For an edit-and-rerun loop, start a development session first:

```bash
antioch services up --watch
```

`services up` builds changed
services, starts their dependencies, waits for health checks, and returns a
service
table. `--watch` is a foreground session that applies `watch` rules while
declared
ports remain reachable. Ctrl-C ends the watcher but leaves containers and
declared ports
running; use `antioch services down` to stop them. A `sync` rule batches file
changes, `sync+exec` runs its command
afterward, `sync+restart` restarts only that service, and `rebuild` rebuilds the
service and its dependents (`manifest.md` owns the decision table). If a
watched service exits, the session fails
loudly; fix the project and run `services up` again.

## One-off probes

```bash
antioch run scripts/evaluate.py -- --episodes 20
```

`FILE` is a project-local Python file. Arguments after `--` reach the script
unchanged. The command has no JSON output because the program owns stdout;
use a scenario when a result must be retained. `--machine MACHINE` pins an
assigned machine, `--timeout` bounds the process, and `--no-stream` opts out of
the default livestream. Check `antioch run --help` for the exact option set.

## Selecting scenarios

Preview the local catalog before dispatching:

```bash
antioch scenario collect --json
```

Selection is flag-based; there is no positional scenario name:

```bash
antioch scenario run --scenario bin_pick --set seed=42
antioch scenario run --tag warehouse --exclude-tag slow
antioch scenario run --path scenarios/warehouse
antioch scenario run --scenario bin_pick --case nominal
```

`--scenario`, `--tag`, `--exclude-tag`, `--path`, and `--case` can be repeated.
A scenario without `--case` runs its declared cases. `--case` and `--set` each
address exactly one `--scenario`, and they cannot be combined. `--machine
MACHINE` selects exact machines for the run; `--machines INTEGER` bounds
parallel use.

## Watching output

Foreground scenario output is a live board and deliberately has no JSON
output. Two escalations exist when the board hides what you need:
`--verbose` relays the process output while filtering Kit engine noise, and
`--raw-logs` relays every byte unfiltered (it implies `--verbose`). Read
durable truth after the run finishes:

```bash
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario logs SCENARIO_RUN_ID
antioch scenario download SCENARIO_RUN_ID
```

## Queueing and reruns

Add `--queue` to a scenario selection to submit it headlessly:

```bash
antioch scenario run --scenario bin_pick --queue --json
```

The submitter builds or resolves every selected service **on the project's
current machine** — `machine checkout` selects that staging machine — adds
the current project source to the simulation image, and retains the
digest-pinned
environment with the run. Antioch then distributes runs across eligible
machines that are independent of the stager. Development `watch` rules and
`ports` tunnels are absent from the queued environment.

Queue flag constraints: do not combine `--queue` with a typed `--stream`,
with `--verbose` or `--raw-logs`, or with `--machine` or `--machines` — each
is refused. `--restart` has no effect on queued work (workers always start a
fresh stack), and `--json` on a dispatch command requires `--queue`. Cancel a
queued or running standalone scenario with
`antioch scenario cancel SCENARIO_RUN_ID --json`; suite members are cancelled
through `suite cancel` (`suites.md`).

Queued runs save their exact digest-pinned environment before Antioch
distributes them. For a single-machine interactive run, Antioch captures the
admitted `sim` container's project workspace at dispatch and then attempts to
save the exact images and source the run used. When that capture and publish
succeeds,
`antioch scenario rerun SCENARIO_RUN_ID` and `antioch suite rerun SUITE_RUN_ID`
queue the completed run again with the same images, parameters, and cases, and
a fresh identity. Multi-machine interactive runs are not currently rerunnable.
Repeat the original command or use `--queue` when the result must be rerunnable.
Antioch explains when an older run or a failed source capture or publish leaves
the environment unavailable.

When no watcher is running, every `run` and scenario or
suite dispatch reconciles the current tree once before it starts. A live watcher
already owns that publication, so edits are never hidden behind a stale upload.

## Streaming and failures

Simulation runs stream by default. `--no-stream` makes a run headless, while an
explicit `--stream` requires the machine's single livestream lease. A busy
lease makes an explicit request fail; an unqualified default can continue
headlessly with a notice. Queued runs are headless by contract.
`antioch services exec` never reserves the livestream — the lease is taken by
default by `antioch run`, `scenario run`, and `suite run`, and declared for a
kernel by `antioch jupyter stream` (`sessions.md`).

If local discovery fails before a machine is requested, check for a simulator
import at module scope and rerun `antioch scenario collect --json`. If a
script's flag is consumed by Antioch, put it after `--` in the `run` command.
