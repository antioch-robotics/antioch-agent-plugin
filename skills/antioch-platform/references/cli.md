# The CLI command map and machine contract

This file is the complete map of the `antioch` CLI: every command group,
every leaf, the global environment variables, the credential store, the
stdout/stderr and exit-code contracts, and the proactive plays agents
underuse. Per-command workflow depth stays in the topical references —
`running.md`, `scenarios.md`, `suites.md`, `machines.md`, `sessions.md`,
`assets.md`, `auth.md`. Use `antioch <command> --help` as the authority for a
detail this map does not carry.

## The tree

Top-level commands, in help order: `run`, `jupyter`, `scenario`, `suite`,
`machine`, `services`, `assets`, `project`, `auth`, `init`. There is no
`workspace`, `session`, `queue`, or top-level `exec` group — queueing is the
`--queue` flag on scenario and suite runs, exec lives at `services exec` and
`machine ssh`, and Mission Control workspaces have no CLI verbs (see
`mission-control.md`). Bare `antioch` and any bare group print help on stdout
and exit 0. An unknown name gets a "did you mean" suggestion.

### Dispatch

| Command | Purpose | Key options | JSON |
|---|---|---|---|
| `run FILE [ARGS]...` | Run one project-local `.py` file on a machine; output relays directly | `--machine`, `--stream/--no-stream`, `--restart`, `--timeout` (900 s), `--verbose` | none — the program owns stdout; exit code is the remote process's |
| `scenario run` | Select and run authored scenarios (flag-based; no positional name) | `--scenario`, `-t/--tag`, `--exclude-tag`, `--path`, `--case`, `--set`, `--machine` (repeatable), `--machines`, `--timeout`, `--stream/--no-stream`, `--restart`, `--verbose`, `--raw-logs`, `--queue` | `--json` requires `--queue`; bare array of queued run records |
| `suite run NAME` | Run one authored suite | same dispatch options as `scenario run` | `--json` requires `--queue`; bare queued suite object |

### Scenario history

| Command | Purpose | Key options | JSON |
|---|---|---|---|
| `scenario list` | Filter organization-shared run history | `-q/--search`, `--scenario`, `--project`, `--suite`, `--suite-run-id`, `--invocation-id`, `--dispatched-from`, `--user`/`--mine`, `-t/--tag`, `--no-suite`, `--param`, `--result`, `--since`/`--until`, `--dispatch`, `--phase`, `--outcome`, `--cursor`, `--limit` (1–200, 50) | `{items, next_cursor}` of run records |
| `scenario show SCENARIO_RUN_ID` | One run: phase, outcome, params, results, checks, and saved artifact names | `--logs`, `--service` (requires `--logs`) | run record |
| `scenario logs SCENARIO_RUN_ID` | Replay captured output — stdout to stdout, stderr to stderr | `--stdout`/`--stderr` (exclusive), `-f/--follow`, `--raw-logs` | `{items, next_cursor: null}` of log entries |
| `scenario download SCENARIO_RUN_ID` | Signed-storage artifact download into `./SCENARIO_RUN_ID/` | `--artifact` (repeatable, logical key), `-o/--output`, `--force` | transfer manifest with per-file `sha256` |
| `scenario cancel SCENARIO_RUN_ID` | Cancel a standalone queued or running scenario run | — | cancellation object with `changed` |
| `scenario rerun SCENARIO_RUN_ID` | Queue a completed run again from its saved environment | — | the queued run record |
| `scenario delete` | Delete standalone runs you own | `--run` (repeatable), `--yes` | requires `--yes` |
| `scenario suggest FIELD` | Distinct stored values and counts for one filter field: `user_email`, `tag`, `suite`, `scenario`, `project`, or `dispatched_from`; optional `PREFIX` positional | `--limit` (1–100, 20) | `{items, next_cursor: null}` |
| `scenario collect` | Local discovery and validation; no network, no machine | optional `PATH` positional | `{items, next_cursor: null}` of definitions |

### Suite history

| Command | Purpose | Key options | JSON |
|---|---|---|---|
| `suite list` | Suite runs; the current project is the default filter inside a project | `--suite`, `--project`, `--all-projects`, `--phase`, `--outcome`, `--user`/`--mine`, `--since`/`--until`, `--dispatch`, `--cursor`, `--limit` | `{items, next_cursor}` of suite records |
| `suite summary` | Named suites with latest-run state | `-q/--search` plus the `list` filters, `--limit` (1–500, 50) | `{items, next_cursor}` of summaries |
| `suite show SUITE_RUN_ID` | One suite run with its member scenario runs | `--suite` (only for a legacy id), `-f/--follow` | suite record; `--follow --json` is NDJSON state frames ending in a `completed` frame |
| `suite cancel SUITE_RUN_ID` | Cancel unclaimed members and signal active processes | — | adds `changed` |
| `suite rerun SUITE_RUN_ID` | Queue a completed suite again | — | bare queued suite object |
| `suite delete` | Delete suite runs AND their member scenario runs | `--run`, `--suite` (both repeatable), `--yes` | requires `--yes` |
| `suite collect` | Local suite expansion exactly as dispatch would see it | — | `{items, next_cursor: null}` |

### Machines

| Command | Purpose | Key options | JSON |
|---|---|---|---|
| `machine list` | Your assignments (personal, never another user's) | `--project`, `--all` | bare array of rows: id, urls, GPU, placement, state, generation, project, `dispatched_from`, `current` |
| `machine status` | One machine plus live process, stream, and container state | `--project`, `--machine` | bare object |
| `machine checkout` | Set the project's current machine (git-checkout analogy); local, allocates nothing; optional `MACHINE` positional | `--none` clears | mutation result; bare invocation prints the current machine id on stdout |
| `machine release` | Release one assignment and stop its work; **idempotent** — already-released is success; optional `MACHINE` positional | `--project`, `--machine` | released assignment |
| `machine ssh [CMD]...` | VM host shell; a one-shot CMD runs without a PTY, exit status is the command's | `--machine` | none — interactive/relayed |
| `machine usage` | Measured GPU machine time over a window | `--since` (default `7d`), `--until`, `--project`, `--user`/`--mine`, `--limit` (1–500, 20) | usage summary |

### Services

| Command | Purpose | Key options | JSON |
|---|---|---|---|
| `services up` | Build changed services, start, wait for health; may allocate | `--profile` (repeatable), `--watch` (foreground; cannot combine with `--json`) | stack state |
| `services down` | Stop and remove project services; requires an assigned machine and never assigns one | — | teardown object |
| `services ps` | Inspect state without allocating | — | stack state |
| `services logs` | Stream raw container bytes; optional `SERVICE` positional | `-f/--follow`, `--tail`, `--since` | none — raw stream |
| `services restart` | Restart in dependency order, no rebuild | `--service` (repeatable) | restarted services |
| `services build` | Build without starting | `--service` (repeatable) | build results |
| `services exec SERVICE CMD...` | Run CMD in a service; unknown options pass through to CMD; exit code is the command's | — (no options of its own) | none |
| `services ssh` | Interactive PTY in a service; optional `SERVICE` positional, default `sim` when present | — | none |
| `services cp SRC DST` | Copy files/directories; exactly one side is `SERVICE:PATH`; a destination ending in `/` is a directory and receives the source basename, while a destination without `/` receives the source contents; symlinks copied as links | — | manifest with `direction`, `size_bytes`, `sha256` |
| `services images` | List retained build products (bare group invocation) | — | build products |
| `services images pull SERVICE` | Export a built service image to the local Docker daemon | `--tag` | export manifest |

### Assets, project, auth

| Command | Purpose | Key options | JSON |
|---|---|---|---|
| `assets list` | Organization assets plus Antioch's shared library | `-q/--search`, `--cursor`, `--limit` (1–200, 50) | `{items, next_cursor}` |
| `assets show ASSET` | One asset with its published versions | `-v/--version` | bare object |
| `assets push PATH` | Publish one immutable version | `-n/--name`, `-v/--version` (**required**), `-d/--description`, `--content-type`, `--preview` | published version |
| `assets pull ASSET` | Signed-storage download | `-v/--version` (latest), `-o/--output`, `--preview`, `--force` | transfer manifest |
| `assets verify ASSET` | Temp-download and integrity check; nonzero exit on failure | `-v/--version` | none |
| `assets repair ASSET` | Remove published versions whose file is missing | `-v/--version` (default: every version) | `{asset, asset_id, removed, kept}` |
| `project current` | Project selected by the working directory | — | bare object; literal `null` outside any project |
| `project list` | Locally known projects | — | bare array |
| `project show` | One local project by name or id; optional `PROJECT` positional | — | bare object |
| `init` | Scaffold a project locally (no prompts, no remote registration); optional `DIRECTORY` positional; refuses a directory already inside a project | `--engine` (`isaac-sim-6.0.1` or `isaac-lab-3.0`; default from the installed extra) | `{project_id, name, engine, path, remote_registered}` |
| `auth login` | Device-code sign-in (works over SSH); refused inside Mission Control | — | identity |
| `auth logout` | Remove the local session, machine capabilities, and SSH keys | — | signed-out state |
| `auth whoami` | Active user and organization; human output on stdout | — | identity |
| `auth switch` | **Interactive only**: numbered organization prompt; no JSON, no flags | — | none |

### Jupyter

All subcommands take `--machine`; `--kernel` is required only when several
kernels are live. JSON outputs are bare objects.

| Command | Purpose | Key options |
|---|---|---|
| `jupyter start` | Start one Isaac kernel; human mode prints the bare kernel id on stdout | `--json` |
| `jupyter cell` | Run one cell and exit; the kernel stays warm; optional `CODE` positional, stdin when absent | `--kernel`, `--timeout` (900 s), `--json` |
| `jupyter kernels` | Live kernels on every assigned machine | `--json` (bare array) |
| `jupyter show KERNEL_ID` | One kernel's state and next-step commands | `--json` |
| `jupyter stop` | Shut one kernel down, releasing GPU memory | `--kernel`, `--json` |
| `jupyter stream` | Put the kernel's Isaac GUI on the machine livestream | `--kernel`, `--json` |
| `jupyter unstream` | Release the stream without stopping the kernel | `--kernel`, `--json` |
| `jupyter lab` | Local JupyterLab against the remote kernel; prints the verified URL | `--no-open`, `--verbose` |

## Global environment variables

| Variable | Meaning |
|---|---|
| `ANTIOCH_ENV` | Deployment profile: `staging` (default) or `prod`. Set it before `auth login` and keep it set — credentials are stored per environment. |
| `ANTIOCH_CONFIG_DIR` | Exact config-directory override. Isolates credentials and allocations per agent or test run without touching the user's store. |
| `XDG_CONFIG_HOME` | Config root when no override; default `~/.config`. |
| `ANTIOCH_WORKSPACE_ID` | Set by Mission Control inside a workspace; switches the dispatch origin to `cloud-workspace`. Not user-set. |
| `ANTIOCH_ENGINE`, `ANTIOCH_SCENARIO_*` | Injected into containers and dispatched processes by Antioch. Never set these. |

## Credential store

Antioch stores credentials and local machine state under
`$ANTIOCH_CONFIG_DIR` or `$XDG_CONFIG_HOME/antioch/<environment>/`. Deployment
profiles use separate directories, so their sessions cannot collide.

## stdout vs stderr

Chrome is never data. Success notes, warnings, spinners, and step lines go to
**stderr**. Human tables and all `--json` payloads go to **stdout**. A
redirected table renders at full width so `grep` and `awk` see every column.
Confirmed one-off data lines on stdout: `machine checkout` with no argument
(the bare machine id), `jupyter start` (the kernel id), `jupyter lab` (the
URL), `auth whoami` (the identity key-values). `scenario logs` replays the
run's stdout to stdout and stderr to stderr.

## JSON shapes

- Listing commands emit `{"items": [...], "next_cursor": <string or null>}`.
  Detail and mutation commands emit a bare object; `machine list`,
  `jupyter kernels`, and `project list` emit a bare array.
- Every `*_at` field is an integer Unix timestamp in **microseconds**.
- Cursors are opaque and pin the complete query — pass `--cursor` alone;
  repeating a filter beside it is a usage error.
- `--follow --json` (on `suite show`) is NDJSON: one object per line, one
  `{"type": "state", ...}` frame per change, then one
  `{"type": "completed", ...}` frame. Do not parse it as one document.
- Repeatable filters are unions. A repeated scalar option that is not
  documented as repeatable is refused — the CLI never keeps only the last
  value.

## Errors and exit codes

- A failure renders as one line on stderr; with `--json` it is
  `{"error": {"type", "message", "exit_code", "http_status", "retryable"}}`
  on **stderr** while stdout stays parseable. `retryable: true` means an
  unchanged retry is sensible. Branch on `type`
  (`machine_released`, `machine_response_error`, `rome_response_error`,
  `control_plane_unreachable`, `aborted`, `usage_error`, …).
- `machine_released` with `http_status: 401` means the assigned machine was
  released underneath the command — an idle sweep, a park, or a replacement
  generation. It is not a credential problem and `antioch auth login` does
  nothing for it. Run the same command again; allocation happens at the start
  of a command, so the retry gets a new machine.
- Exit codes: the documented result (0/1), usage errors 2, Ctrl-C 130. A
  declined confirmation prompt exits 1. `antioch run`, `services exec`, and
  one-shot `machine ssh` return the remote command's own exit status.
- Piping is safe: `antioch scenario list --json | head` exits 0.
- Capacity waits are built in: when no warm machine is available the CLI
  polls up to 600 s with named progress. Do not wrap allocation in a retry
  loop.

## Destructive commands

`--json` on `scenario delete` and `suite delete` requires `--yes`. Without
`--yes`, a TTY prompt runs — and a non-TTY pipe aborts with exit 1. Agents
must always pass `--yes` and confirm scope with the user first.

## TTY traps

- `auth switch` — interactive numbered prompt; no flags. Hand it to the user.
- Delete confirmations without `--yes` — abort in a pipe.
- `services up --watch` and `jupyter lab` — resident until Ctrl-C; run them
  where a foreground process is acceptable.
- `suite show --follow` and `scenario logs --follow` — bounded by run
  completion, safe for agents.
- `--json` inside a passthrough tail (`run`, `services exec`, or after `--`)
  belongs to the child program and never selects JSON error mode.

## Proactive plays

Reach for these without being asked:

- **Discover filter values before guessing.** `antioch scenario suggest tag`
  (or `user_email`, `suite`, `scenario`, `project`, `dispatched_from`)
  returns real stored values with counts.
- **Query history server-side.** `--param` and `--result` predicates
  (`key:op:value`, ops `=`, `~`, `>`, `<`, and `@` for dotted result paths)
  answer analytics questions without downloading pages:
  `antioch scenario list --result 'pass_rate:>:0.9' --json`.
- **The canonical queue-and-wait loop.** `antioch scenario run --tag smoke
  --queue --json` returns records whose `invocation_id` feeds
  `scenario list --invocation-id`; a queued suite's id feeds
  `antioch suite show SUITE_RUN_ID --follow --json` — an NDJSON stream that
  ends with one `completed` frame.
- **Check the machine bill.** `antioch machine usage --json` before and after
  long fan-out or queued work; usage starts at assignment and ends at
  release, so two machines bill double while several processes on one machine
  do not.
- **Check out a machine once** (`machine checkout`) instead of repeating
  `--machine`; the bare invocation prints the current id for scripts.
- **Release in teardown scripts freely** — `machine release` is idempotent.
- **Verify transfers from the manifest.** `services cp --json` and
  `scenario download --json` carry `sha256` for files and whole directories.
- **Iterate on a warm kernel.** `jupyter cell` keeps the kernel (and loaded
  stage) alive between cells; code can come from stdin or a heredoc.
- **Fan out precisely.** `--machines N` sets a ceiling; repeated `--machine`
  names the exact set. Only run commands accept a machine list.
- **Recreate services precisely.** Bare `--restart` recreates everything the
  dispatch uses; `--restart SERVICE` (repeatable) recreates exactly those.
- **Preserve a built image before release** with
  `antioch services images pull SERVICE` — a released generation's
  credentials are useless by design, so pull while the assignment is live.
