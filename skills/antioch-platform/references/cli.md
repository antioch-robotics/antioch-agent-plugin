# CLI workflows

Use this guide to choose a command sequence. Use the installed CLI help for
the current command surface and options.

```bash
antioch --help
antioch project --help
antioch session --help
antioch service --help
antioch service exec --help
antioch jupyter --help
antioch scenario --help
antioch suite --help
```

The top-level groups cover scenarios, suites, sessions, services, Jupyter,
assets, projects, setup, and authentication. `antioch service exec` runs one command
in a service of a live session and starts a session when none is live.
`antioch init` creates a local project. Use each group's `--help` for the
current surface.

## Install and update local tools

```bash
antioch setup --dry-run
antioch setup
ANTIOCH_ENV=staging antioch setup
antioch project update --dry-run
antioch project update
```

Use only the operation the user requested. `antioch setup` selects one
deployment's SDK and verified plugin pair for this computer. Production is the
default; `ANTIOCH_ENV=staging` selects staging, as it does for every antioch
command. Do not edit shell files yourself. Staging retains signed private
delivery and existing provider
credentials. Production needs no provider login. Older deployments or SDKs may
lack public release facts or plugin binding; report the rollout gap, not a
guessed latest pair. A fresh public `uvx --from antioch-sim antioch setup` is
usable only after a setup-capable SDK is normally published. The older SDK-only
bootstrap is `uv tool install --python 3.12 antioch-sim`, not paired setup
acceptance.

Setup configures every Claude Code or Codex client on PATH; a host without an
agent gets only the SDK. After installing, it always checks existing sign-in
and a real Research call but never signs in; a failed check exits 1 after
reporting every component and names the next step, such as
`ANTIOCH_ENV=staging antioch auth login`. `--dry-run` does not install,
configure, or check; client inspection can write its ordinary logs. Setup never
edits a shell profile or a project. The retired `antioch plugin install` only
emits DEP005 and performs no setup.

`antioch project update` leaves this computer unchanged. It preserves engine
families, dependency extras and markers, runs native uv lock/sync, and builds
changed literal Dockerfile pins through the existing revision path. Native uv
work is skipped only when the exact installed SDK wheel matches and the plan
leaves `pyproject.toml` unchanged. Use `--no-build` if the user excluded
builds. Normal build and package-manager operations still never rewrite
Dockerfiles. The update does not select a new session revision, start compute,
or restart processes; a live session keeps its previous image, so run
`antioch session new` afterwards to work on the updated project. Unsupported
ARG/digest/symlink inputs stop the plan; an
optional/group-only SDK needs explicit `--python PATH`. A project without its
own SDK environment builds in the SDK `antioch setup` installed. Do not enable
every project extra to work around a refused selection.

Both commands emit the same components in text and JSON. `mode: plan` means a
dry run; `mode: apply` means installation or update. Setup's `sdk`,
`plugin_tree`, and `agents` entries are `current` (verified before the call),
`planned` (needed but not applied), or `changed` (applied and verified);
`plugin_tree` is null without an agent. `project_sdk` names the current
project's own `.venv` interpreter and whether it is `current`; it is null
outside a project that owns `antioch-sim`, and a project that is behind needs
`antioch project update`. `authentication` and `research` carry a
`state` of `ok`, `failed`, or `planned` and a `detail`. `release`,
`sdk_version`, and `plugin_version` identify the pair. The project update
reports `root`, `sdk` (null when the project owns no environment), `files`
with project-relative paths including observed lock changes after sync, and
`sync` and `build` actions of `planned`, `completed`, or `skipped`; completed
builds include revision IDs. Do not infer installation from mode alone. There is
no aggregate configured flag or subprocess command list.

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
working directory and lists every service: its source (`build` or `image`),
the GPU class of the machine it lands on, its CPU and memory requests, and the
profiles that activate it. Use `antioch project list` when a list-shaped
response is more useful; there is no separate current-project command.

Run these commands from the directory that contains `antioch.yaml`. Use the
switch command when the account has more than one organization; `ORG` must be
the exact organization ID.

Builds resolve every selected service to an immutable registry digest and
finalize one project revision. A local build is skipped only when its
content-derived build key resolves to an immutable registry digest with
matching labels. Inspect the current project's revision history without
starting a session:

```bash
antioch project revision list
antioch project revision show REVISION
antioch project revision tag candidate REVISION
```

A revision is an immutable service graph; `antioch project revision list --json`
carries the image digest each service pins. A tag is a movable name for an
existing revision; moving it does not rebuild or mutate that revision.

## Use an interactive session

```bash
antioch session new
antioch session list
antioch service exec python src/main.py
antioch service exec --service autonomy -- ros2 topic list
antioch session status
```

`service exec` runs literal argv in the simulator service, or the only active
service when there is no simulator. `--service` selects another service.
It uses the project's live session and starts one when none is live. Put
Antioch options before CMD; every later token belongs to that command. There
is no top-level `run` alias. A project has one live interactive session and
every command acts on the project of the current directory, so nothing takes
a session selector. Direct access commands operate on the existing session:

```bash
antioch service exec --service sim -- nvidia-smi
antioch service logs SERVICE...
antioch service shell
antioch service cp sim:/workspace/project/result.png ./result.png
```

Use `antioch session release` when the session is no longer needed. Its
services and temporary files go away.

## Use Jupyter

```bash
antioch jupyter lab
antioch jupyter cell '1 + 1'
antioch jupyter cell --stream 'import antioch; antioch.start_simulation()'
antioch jupyter lab --stop
```

Jupyter uses the project's interactive session and never starts one.
`jupyter cell` starts a kernel
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

By default, scenario and suite work runs serially on the project's interactive
session and follows until completion. `--no-follow` returns after admission
without changing its session or stream intent. Use `--detach` for headless
background compute; `--follow` can wait for either mode. Background fan-out is
automatic within your quota and available capacity. Scenario and suite records keep their progress and results
independently of the sessions that execute them.

Read saved results:

```bash
antioch scenario show SCENARIO_RUN_ID
antioch scenario logs SCENARIO_RUN_ID
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
antioch asset list -q "mobile robot" --json
antioch asset show robots/example
antioch asset pull robots/example --version v1 --output ./assets/example
antioch asset push ./assets/example.usdz --name robots/example --version v2
antioch asset verify robots/example --version v2
```

Search before creating an asset: every `-q` word must appear in the name or
description. Use `antioch asset --help` for paging and repair workflows.

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
antioch service ports --help
antioch jupyter lab --help
antioch scenario show --help
antioch suite show --help
```
