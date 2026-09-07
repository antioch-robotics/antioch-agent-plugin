# CLI workflows

Use this guide to choose a command sequence. Use the installed CLI help for
the current command surface and options.

```bash
antioch --help
antioch project --help
antioch session --help
antioch services --help
antioch services exec --help
antioch jupyter --help
antioch scenario --help
antioch suite --help
```

The top-level groups cover scenarios, suites, sessions, services, Jupyter,
assets, projects, and authentication. `antioch services exec` runs one command
in a service of a live session and starts a session when none is live.
`antioch init` creates a local project. Use each group's `--help` for the
current surface.

## Start a project

```bash
antioch auth login
antioch auth whoami
antioch auth switch --org ORG --json
antioch init
antioch project show
antioch project build
```

`antioch project show` without an argument reads the project selected by the
working directory. Use `antioch project list` when a list-shaped response is
more useful; there is no separate current-project command.

Run these commands from the directory that contains `antioch.yaml`. Use the
switch command when the account has more than one organization; `ORG` must be
the exact organization ID.

Builds resolve every selected service to an immutable registry digest and
finalize one project revision. A local build is skipped only when its
content-derived build key resolves to an immutable registry digest with
matching labels. Inspect the project revision history without starting a
session:

```bash
antioch project revision list
antioch project revision show REVISION
antioch project revision tag candidate REVISION
antioch project services versions sim
```

A revision is an immutable service graph. A tag is a movable name for an
existing revision; moving it does not rebuild or mutate that revision.

## Use an interactive session

```bash
antioch session start
antioch session list
antioch services exec python src/main.py
antioch services exec --service autonomy -- ros2 topic list
antioch session status --session SESSION
```

`services exec` runs literal argv in the simulator service, or the only active
service when there is no simulator. `--service` selects another service.
It starts or reuses a compatible session. Put Antioch options before CMD;
every later token belongs to that command. There is no top-level `run` alias.
Commands that select a session accept `--session SESSION`;
without it the CLI uses the session it last used in this worktree, then the
sole live interactive session. Direct access commands operate on an existing
session:

```bash
antioch services exec --session SESSION --service sim -- nvidia-smi
antioch services logs --session SESSION SERVICE...
antioch shell --session SESSION
antioch services cp sim:/workspace/project/result.png ./result.png
```

Use `antioch session stop --session SESSION` when the session is no longer
needed. Its services and temporary files go away.

## Use Jupyter

```bash
antioch jupyter lab
antioch jupyter cell '1 + 1'
antioch jupyter cell --stream 'import antioch; antioch.start_simulation()'
antioch jupyter lab --stop
```

Jupyter uses the selected interactive session and never starts one; add
`--session SESSION` to name another session. `jupyter cell` starts a kernel
when none is live; JupyterLab's own controls manage kernels. Use
`antioch jupyter lab --stop` to stop JupyterLab.

## Run recorded evaluations

Collect definitions locally before submission:

```bash
antioch scenario collect --json
antioch suite collect --json
```

Submit and follow work:

```bash
antioch scenario run --scenario falling_cube
antioch suite run acceptance
```

By default, scenario and suite work runs serially on the selected interactive
session and stays attached until it finishes. Use `--detach` for unattended
work; `--parallel N` sets a detached suite's maximum background session
fan-out. Scenario and suite records keep their progress and results
independently of the sessions that execute them.

Read saved results:

```bash
antioch scenario show SCENARIO_RUN_ID
antioch scenario show SCENARIO_RUN_ID --logs
antioch scenario download SCENARIO_RUN_ID
antioch suite show SUITE_RUN_ID
antioch suite summary
```

Use `antioch scenario rerun SCENARIO_RUN_ID` or
`antioch suite rerun SUITE_RUN_ID` to repeat saved service images and exact
inputs under a new run ID. A rerun can still have a different outcome or
duration when scheduling, capacity, simulator timing, or external assets
differ.

## Work with assets

```bash
antioch assets list -q "mobile robot" --json
antioch assets show robots/example
antioch assets pull robots/example --version v1 --output ./assets/example
antioch assets push ./assets/example.usdz --name robots/example --version v2
antioch assets verify robots/example --version v2
```

Search before creating an asset: every `-q` word must appear in the name or
description. Use `antioch assets --help` for paging and repair workflows.

## Use structured output

Finite commands support JSON unless their real output is a byte stream, a
process stream, an interactive terminal, or a local URL. Add `--json` when an
agent or script needs stable fields. Followed JSON output is line-delimited so
each state change is one document.

Commands write structured data to stdout and progress or errors to stderr.
Structured errors include a `retryable` verdict. Do not parse display text or
infer success from submission alone.

## Confirm before automation

The CLI help is the final contract. Inspect the exact leaf command before you
write a script:

```bash
antioch services ports --help
antioch jupyter lab --help
antioch scenario show --help
antioch suite show --help
```
