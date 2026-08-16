# Running code and scenarios

Antioch runs simulation code in two main ways. `antioch run` is a one-off probe whose
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
output. Use `--verbose` to replace the board with the complete native process
stream. Read the saved result after the run finishes:

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

The submitter builds or finds every selected service on the project's checked-out
machine, its sole assignment, or a new assignment, in that order. Queued dispatch
does not accept `--machine`; use `machine checkout` when you need to select one
of several assigned machines. Antioch adds the current project files to the simulation image, saves
the exact images and run inputs, and then distributes the work across other
eligible machines. Development `watch` rules and `ports` connections are not
included in queued runs.

Queue flag constraints: do not combine `--queue` with a typed `--stream`,
with `--verbose`, or with `--machine` or `--machines` — each
is refused. `--restart` has no effect on queued work (workers always start a
fresh stack), and `--json` on a dispatch command requires `--queue`. Cancel a
queued or running standalone scenario with
`antioch scenario cancel SCENARIO_RUN_ID --json`; suite members are cancelled
through `suite cancel` (`suites.md`).

Queued runs save their exact service images, project files, and inputs before
Antioch distributes them. For a single-machine interactive run, Antioch captures
the files in the `sim` container and then attempts to save the exact images and
files the run used. When that capture and publish
succeeds,
`antioch scenario rerun SCENARIO_RUN_ID` and `antioch suite rerun SUITE_RUN_ID`
queue the completed run again with the same images, files, parameters, and
cases, under a new run ID. Multi-machine interactive runs are not currently rerunnable.
Repeat the original command or use `--queue` when the result must be rerunnable.
Antioch explains when an older run or a failed source capture or publish leaves
the environment unavailable.

When no watcher is running, every script, scenario, and suite run syncs the
latest project files once before it starts. A live watcher already keeps those
files current.

## Streaming and failures

Simulation runs stream by default. `--no-stream` makes a run headless, while an
explicit `--stream` requires the machine's single livestream lease. A busy
lease makes an explicit request fail; an unqualified default can continue
headlessly with a notice. Queued runs are always headless.
`antioch services exec` never reserves the livestream — the lease is taken by
default by `antioch run`, `scenario run`, and `suite run`, and declared for a
kernel by `antioch jupyter stream` (`sessions.md`).

If local discovery fails before a machine is requested, check for a simulator
import at module scope and rerun `antioch scenario collect --json`. If a
script's flag is consumed by Antioch, put it after `--` in the `run` command.
