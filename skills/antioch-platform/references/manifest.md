# `antioch.yaml` — the complete manifest reference

This file lists the complete schema for the project manifest. It covers
every accepted key, every rejected key with its remedy, the `sim` service
contract, build and image mechanics, watch rules, ports, and the settings
Antioch injects at runtime. An agent can author any valid manifest from this
file alone. Workflow context — creating a project, choosing the simulation image,
private registries — lives in `environment.md`.

## Ground rules

- The manifest file is exactly `antioch.yaml`. The CLI finds it by walking up
  from the current directory, stopping at the first `.git` boundary or the
  filesystem root. A missing manifest points at `antioch init`.
- The document must be a YAML mapping, parsed as YAML 1.2 core: duplicate
  mapping keys are rejected, `yes`/`no` are strings rather than booleans,
  `012` is not octal, and ISO dates stay strings.
- A `compose.yaml` or `compose.override.yaml` beside the manifest is a hard
  error. Declare every project service under `services` in `antioch.yaml`.
- The schema is v4 only. Any other `schema_version` fails at load time.
- Unknown fields are rejected everywhere — top level, service, build,
  healthcheck, watch. The error names the field and lists the supported keys.
- Removed legacy top-level fields fail with a dedicated remedy: `root` is
  derived from the manifest path; `engine` moved to the built sim image's
  labels; `image` belongs in a Dockerfile; `sync` is owned by `.dockerignore`;
  `env` belongs on a service's `environment`; a top-level `sim:` section must
  be declared as `services.sim`.
- Every schema validation failure renders as
  `invalid '<path>/antioch.yaml': <field.path>: <message>`.

## Top-level keys

| Key | Type | Default | Required | Semantics |
|---|---|---|---|---|
| `schema_version` | literal `4` | `4` | no | Only 4 is accepted. |
| `name` | non-blank string | — | **yes** | Human-readable project name; source of the derived id. |
| `id` | lowercase slug: letters, digits, hyphens; no leading or trailing hyphen; max 63 chars | slugified `name` | no | Immutable project identity. It binds machine assignments and the Docker project label `antioch-<id>`. Changing it mid-session aborts a watch redeploy. |
| `scenario_paths` | list of project-relative POSIX paths (no `..`, no leading `/`, no `\`; a trailing `/` is stripped) | unset | no | Explicit scenario discovery scope. When omitted, discovery scans the sim build context — or the project root when `services.sim` is image-only or absent — for `.py` files, pruning hidden, cache, and virtualenv directories and applying `.dockerignore` plus `.gitignore`. |
| `suites` | map name → suite | `{}` | no | Named declarative selections, run with `antioch suite run NAME`. |
| `services` | map name → service | — | **yes, at least one service** | Every container in the project. The optional name `sim` identifies the simulation service. |

Service names match `^[A-Za-z0-9][A-Za-z0-9_.-]+$` (at least two characters).
Container names become `antioch-<project-id>-<service>`.

## Suites and selections

A suite has an optional `description` and a required `select` list with at
least one clause. Fields inside one clause narrow together (AND); clauses are
unioned in authored order.

| Clause field | Semantics |
|---|---|
| `paths` | Project-relative files or directories. |
| `scenarios` | Exact scenario names. |
| `cases` | Exact case ids; a scenario without a matching declared case is excluded. |
| `tags` | ALL listed tags must be present on the scenario or case. |
| `exclude_tags` | ANY listed tag disqualifies. |

Tags are 1–128 characters with no commas and no surrounding whitespace.

## Service keys

The kept vocabulary is deliberately familiar from
[Docker's Compose file reference](https://docs.docker.com/reference/compose-file/);
Docker runs underneath Antioch. The 17 supported service keys:

| Key | Type | Default | When to use |
|---|---|---|---|
| `build` | mapping or string shorthand | unset | Build the image on the assigned machine from a Dockerfile. One of `build`/`image` is required. See "Build" below. |
| `image` | string | unset | Use a registry image directly. For `sim`, use `antioch-engine/<engine>` (add `:<sdk-version>` only to pin one release); other services can use any registry image. |
| `command` | string (shlex-split) or list | unset; sim gets `["sleep", "infinity"]` injected | Container command override. |
| `entrypoint` | string (shlex-split) or list | unset | Entrypoint override. |
| `environment` | map or `NAME=VALUE` list | `{}` | Static service environment. Names match `^[A-Za-z_][A-Za-z0-9_]*$`. A `null` map value is rejected — write an explicit `KEY: ""`; there is no host-environment inheritance. Names starting `ANTIOCH_` are reserved for injected values. Booleans lower to `true`/`false` strings. |
| `working_dir` | non-blank string | unset | Working-directory override. |
| `depends_on` | list shorthand, or map name → condition | `{}` | Start ordering and readiness gates. Conditions: `service_started` (the default), `service_healthy`, `service_completed_successfully`. Unknown dependencies and cycles fail at load or plan time. |
| `healthcheck` | mapping | unset | Container health probe; see "Healthcheck" below. |
| `profiles` | list of names | `()` | Make a service optional: it starts only when `--profile` (or a scenario's `profile=`) names one of its profiles. Forbidden on `sim`. |
| `labels` | map or `NAME=VALUE` list | `{}` | Docker labels. The `antioch.*` and `com.docker.*` namespaces are reserved. |
| `ports` | int, `"PORT"`, `"LOCAL:REMOTE"`, or mapping; single item auto-wraps into a list | `()` | Authenticated local SSH tunnels to the machine's host network — NOT Docker published ports. See "Ports" below. |
| `watch` | rule mapping or list of rules | `()` | Development file rules; see "Watch" below. |
| `network_mode` | `host`, `none`, `bridge`, `service:NAME`, `container:NAME` | **`host`** | Network namespace. Keep `host` unless the service needs isolation. |
| `ipc` | `host`, `none`, `private`, `shareable`, `service:NAME`, `container:NAME` | **`host`** | IPC namespace; same reference forms as `network_mode`. |
| `cpus` | number > 0 | unset (no limit) | Docker `--cpus` limit for an auxiliary service. |
| `mem_limit` | int bytes or `^\d+(\.\d+)?[bkmg]?$` (case lowered) | unset (no limit) | Docker `--memory`, such as `512m`. |
| `privileged` | bool | **`true`** | Docker `--privileged`. Set `privileged: false` on auxiliary services that do not need device access. `sim` must keep `true`. |

Cross-field rules:

- A service must declare `build` or `image`; declaring neither fails.
- `ports` require `network_mode: host` — a tunnel targets the host network,
  so a non-host mode makes it unreachable and the manifest is rejected.

## Rejected service keys and their remedies

Each fails with `service field '<key>' is not supported — <message>`:

| Key | Remedy |
|---|---|
| `networks` | User-defined networks are not supported; use `network_mode` for namespace sharing. |
| `volumes` | Declare durable output under `/workspace/output`; named volumes are not supported. |
| `restart` | Antioch fails loud — fix the service, then redeploy with `antioch services up`. |
| `deploy` | Deployment settings are not part of the dev subset; redeploy with `antioch services up`. |
| `scale`, `replicas` | Scaling is not part of the dev subset; run one service declaration. |
| `gpus`, `init` | Already injected on every service. |
| `container_name` | Container names are derived from the project and service. |
| `extends`, `include` | One `antioch.yaml` owns the complete stack. |
| `secrets`, `configs` | Declare ordinary environment values; secret and config mounts are not supported. |
| `develop` | Declare watch rules under `watch:` on the service. |

Any other unknown key fails with the supported-key list.

## Rules for the `sim` service

The name `sim` identifies the simulation service. It is **optional**: a service-only
stack (viewer, API, ROS tooling) is valid. Commands that run Isaac code —
`antioch run`, scenario and suite dispatch, Jupyter kernels — refuse a stack
without it: "this stack has no sim service; declare services.sim in
antioch.yaml". When `sim` is present:

1. It must keep `privileged: true`, `network_mode: host`, and `ipc: host` —
   Antioch process control, streaming, and DDS depend on them.
2. `profiles` is forbidden on it; `sim` is always active.
3. It cannot declare both `build` and `image` (other services can; the build
   result wins there).
4. Its `image`, when used, must be an Antioch simulation image:
   `antioch-engine/<engine>`. The engine is `isaac-sim-6.0.1` or
   `isaac-lab-3.0`. Without a tag, runs use the SDK release installed with the
   CLI; an explicit `:<sdk-version>` tag must use semantic versioning. The
   removed `antioch-sim` image names fail.
5. With `build`, exactly one Dockerfile `FROM` line must use a versioned
   `antioch-engine/<engine>:<sdk-version>` image. That value chooses the cloud
   engine and SDK. A `FROM` line without the SDK tag fails validation.
6. With no `command`, the loader injects `sleep infinity` so the container
   idles for exec-style dispatch.
7. No other service may use an engine image — "name the engine service
   'sim'".

## Healthcheck

| Field | Type | Default | Semantics |
|---|---|---|---|
| `test` | string (becomes `CMD-SHELL`) or list starting `CMD`, `CMD-SHELL`, or `NONE` | image default | `NONE` must be alone and excludes the timing fields. |
| `interval`, `timeout`, `start_period`, `start_interval` | number (seconds) or Compose duration string such as `500ms`, `2s`, `1m` | Docker defaults | Finite and non-negative; lowered to nanoseconds. |
| `retries` | int ≥ 0 | Docker default | Failures before unhealthy. |
| `disable` | bool | `false` | `true` disables the image's own healthcheck; cannot combine with `test`, timing, or `retries`. |

Healthchecks (and `antioch services exec`) run WITHOUT the image entrypoint.
On a stock ROS image, source the setup file inside the test command:
`test: ["CMD", "bash", "-c", "source /opt/ros/jazzy/setup.bash && ros2 topic list"]`.

## Ports

A port declaration is an authenticated SSH tunnel from the authoring
computer's loopback to the assigned machine's host network. It is not a
Docker port mapping and there is no service-name network.

| Field | Type | Semantics |
|---|---|---|
| `target` | int 1–65535, required | Port on the machine's host network. |
| `published` | int 1–65535 | Local loopback port; defaults to `target`. |
| `name` | string | Author-facing tunnel name in the CLI output. |

Short forms: `8080` (published = target), `"8080"`, `"9090:8080"`
(LOCAL:REMOTE). Distinct services must publish distinct local ports. A
detached per-project supervisor keeps declared ports open after
`antioch services up` returns, until `services down` or assignment loss.
Ports and watch rules are development-session state: they are never part of a
queued or saved run environment.

## Networking and namespaces

- `network_mode` values: `host` (default), `none`, `bridge`, plus the
  reference forms `service:NAME` and `container:NAME`. `ipc` values: `host`
  (default), `none`, `private`, `shareable`, plus the same reference forms.
  The value sets differ — `private` is not a valid `network_mode` and
  `bridge` is not a valid `ipc`.
- `service:NAME` shares the named project service's namespace and adds an
  implicit start-after dependency on it.
- `container:NAME` points at a machine-local container outside the project.
  It is rejected from queued and saved run environments because it cannot
  travel to another machine.
- Host networking and IPC on every service by default is what makes ROS 2
  DDS discovery work through localhost with no discovery server.

## Injected runtime settings

There is no authorable restart policy or GPU setting. Antioch injects on
every container:

- `--gpus all`, `--init`, and restart policy `no` — a service that dies stays
  dead until you fix it and redeploy; the diagnostics say so.
- The container name `antioch-<project-id>-<service>` and the reserved
  `antioch.project`, `antioch.service`, `antioch.config-hash` labels
  (plus `antioch.sim` on the sim container).
- A host output directory bound read-write at `/workspace/output`.
- For dispatched processes: `ANTIOCH_PROJECT_DIR=/workspace/project`, a
  matching `PYTHONPATH`, and per-process identity and stream variables — all
  under the reserved `ANTIOCH_` prefix, which is why authored `ANTIOCH_*`
  environment names are rejected. The working directory is
  `/workspace/project`.

The CLI's `--restart` flag on `antioch run`, `antioch scenario run`, and
`antioch suite run` is unrelated to Docker restart policy: it recreates
services once before dispatch.

## Build

`build` accepts a string shorthand (`build: .` means context `"."`) or a
mapping with exactly three fields — there is no `target`, `cache_from`, or
`platforms` at this surface:

| Field | Default | Semantics |
|---|---|---|
| `context` | `"."` | Build context, resolved against the project root. |
| `dockerfile` | `"Dockerfile"` | Relative to the context; must stay inside it and must exist. |
| `args` | `{}` | Docker build args; also substituted into `FROM` lines for engine detection. |

How builds run:

- Builds run on the assigned GPU machine's Docker Engine. The CLI streams
  the context as a deterministic tar honoring `.dockerignore` in the context
  root; no local Docker is required.
- A content hash over the context, Dockerfile, ignore rules, build args, and
  resolved base images labels the result. An unchanged context reuses the
  image with zero upload ("context unchanged; reusing image …").
- When the Dockerfile `FROM` names a versioned Antioch simulation image, the machine prepares
  that engine layer first.
- A build stream idle for 30 minutes times out; rerun to reuse completed
  layers from cache.

How images reach machines:

- **Interactive:** built directly on the machine, or pulled by the machine
  from the authored registry using your local Docker credential
  (`auths`, `credsStore`, `credHelpers`), sent only with that pull.
- **Queued:** staging resolves every service on the project's current
  machine, bakes the project source into the sim image, and pushes
  copies identified by exact image digests into your organization's registry.
  Queued environments require `ref@sha256:<digest>` images; a mutable tag or local-only image
  cannot be queued.

## Watch

Watch rules live under each service's `watch:` key. A single rule mapping may
be written without a list. Fields:

| Field | Required | Semantics |
|---|---|---|
| `action` | yes | One of `sync`, `sync+exec`, `sync+restart`, `rebuild`. |
| `path` | yes | Watched project-relative root; must not escape the project. |
| `target` | for all `sync*` actions; forbidden for `rebuild` | Destination directory inside the container. |
| `exec` | for `sync+exec` only; forbidden otherwise | Command to run after each sync (string is shlex-split). |
| `ignore` | no | Rule-local gitignore-style patterns. |

Choosing the action:

- **`sync`** — files the next process reads from disk: Python source,
  configs, assets. The next `antioch run` or scenario dispatch sees them.
- **`sync+exec`** — a long-lived service needs a command after files land,
  such as regenerating protobuf stubs. The command fails loudly on nonzero
  exit.
- **`sync+restart`** — a long-lived service reads its inputs only at start;
  the rule restarts only that container.
- **`rebuild`** — inputs baked into the image: `Dockerfile`,
  `pyproject.toml`, `uv.lock`. A synced file cannot change installed
  dependencies or image layers. Rebuild waits for live run processes to
  finish, rebuilds the service, and converges its dependents.

Mechanism facts that matter:

- The session watches file events (no polling). Deletions propagate; changed
  files ship as a tar archive directly into the container.
- The initial sync owns the entire target subtree: it removes the
  target before uploading, so stale files never survive a new session.
- `.gitignore`, `.dockerignore`, and the rule's `ignore` list are three
  independent boundaries. A negation (`!pattern`) applies only within its own
  group and cannot re-include a path another group excluded.
- One watch session per project (a lock enforces it). Editing `antioch.yaml`
  during a session is inert for pinned rules — the session prints a notice to
  rerun `antioch services up` and restart the watcher.
- Without a resident watcher, every dispatch applies each sync rule once
  before it starts. With one, dispatch verifies source freshness against the
  container tree and refuses when drift persists — check the watch terminal.

## Example manifests

Minimal image-only sim project (the `antioch init` scaffold shape):

```yaml
id: "warehouse-sim"
name: "Warehouse Sim"
scenario_paths: ["src"]

services:
  sim:
    image: "antioch-engine/isaac-sim-6.0.1"
    watch:
      - action: sync
        path: .
        target: /workspace/project

suites:
  smoke:
    description: "Fast engine smoke checks"
    select:
      - tags: ["smoke"]
```

Dockerfile-built sim gated on a healthy ROS service:

```yaml
name: Pick and Place
# id derived: "pick-and-place"

services:
  sim:
    build: .    # context ".", dockerfile "Dockerfile";
                # FROM antioch-engine/isaac-sim-6.0.1:0.3.36 selects the engine and SDK
    environment:
      ROS_DOMAIN_ID: "7"
    depends_on:
      ros:
        condition: service_healthy
    watch:
      - action: sync
        path: src
        target: /workspace/project/src
        ignore: ["__pycache__/"]
      - action: rebuild
        path: pyproject.toml

  ros:
    image: ros:jazzy
    healthcheck:
      test: ["CMD", "bash", "-c", "source /opt/ros/jazzy/setup.bash && ros2 topic list"]
      interval: 2s
      timeout: 5s
      retries: 15
```

Service-only stack — valid with no `sim`; Isaac commands refuse it:

```yaml
name: Telemetry Viewer

services:
  viewer:
    image: python:3.12-slim
    command: ["python", "-m", "http.server", "8080", "--bind", "0.0.0.0"]
    working_dir: /workspace/project
    ports:
      - name: viewer
        published: 8080
        target: 8080          # network_mode defaults to host, so ports are legal
    watch:
      - action: sync
        path: src
        target: /workspace/project/src
```

Full-featured stack — profiles, limits, namespaces, all watch actions, port
short forms:

```yaml
schema_version: 4
id: fleet-lab
name: Fleet Lab
scenario_paths: ["scenarios", "src/sim"]

services:
  sim:
    build:
      context: .
      dockerfile: docker/sim.Dockerfile
      args:
        EXTRA_INDEX: "https://packages.example.com/simple"
    environment:            # list form is also accepted
      - ROS_DOMAIN_ID=7
      - HEADLESS=true
    depends_on: [bridge]    # list shorthand means condition service_started
    watch:
      - action: sync+restart
        path: config
        target: /workspace/project/config
      - action: sync+exec
        path: proto
        target: /workspace/project/proto
        exec: "python -m grpc_tools.protoc --help"   # string is shlex-split
      - action: rebuild
        path: uv.lock

  bridge:
    image: ghcr.io/acme/dds-bridge:2026.08
    entrypoint: ["/entry.sh"]
    labels:
      team: robotics        # antioch.* / com.docker.* are rejected
    healthcheck:
      test: nc -z localhost 7400     # string form becomes CMD-SHELL
      interval: 500ms
      start_period: 10s
    ports:
      - 7400                # int short form: published == target
      - "9090:8080"         # LOCAL:REMOTE string form

  worker:
    image: alpine:3.20
    command: ["sleep", "infinity"]
    cpus: 0.5
    mem_limit: 128m
    privileged: false
    network_mode: service:bridge   # implicit start-after-bridge edge
    ipc: private

  perception-debugger:
    image: ghcr.io/acme/perception-debugger:2026.08
    profiles: ["perception"]       # active only with --profile perception
    depends_on:
      bridge: service_started      # string-condition shorthand

suites:
  acceptance:
    description: "Release gate"
    select:
      - paths: ["scenarios/acceptance"]
        tags: ["release"]
        exclude_tags: ["flaky"]
      - scenarios: ["bin_pick"]
        cases: ["seed-42"]
```

## Ordering, waves, and teardown

Antioch validates the FULL authored graph, including inactive profiles — a
typo hidden behind an inactive profile fails at plan time. Active services
(sim, unprofiled, and enabled profiles) start in parallel dependency waves;
teardown reverses the start order. A cycle names its members and removes no
machine.
