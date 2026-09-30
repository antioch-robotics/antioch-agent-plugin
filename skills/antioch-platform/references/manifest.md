# The project manifest

`antioch.yaml` is a closed, Compose-inspired schema for identity, services,
scenario discovery, source sync, and suites. Start from `antioch init` and
validate with the installed `ProjectManifest` model; unknown fields are
rejected.

## Services and images

```yaml
schema_version: 6
id: warehouse-sim-0123456789abcdef
name: warehouse-sim
scenario_paths: ["src/scenarios.py"]
services:
  sim:
    build:
      context: .
      dockerfile: Dockerfile
    resources:
      gpu: rtx-pro-6000
    sync:
      - source: .
        target: /workspace/project
```

Preserve an existing ID. A top-level `organization_id: org_...` binds the
checkout to one organization: remote commands refuse while another is signed
in. At least one service is required, each with exactly one `image` or
`build`. Antioch image metadata identifies scenario-capable services, not
service names; with several eligible services, select the target with
`service=` in the scenario decorator.

Supporting services can declare commands, environment, working directory, profiles, dependencies, health checks, resources, restart policy, routes, and sync mappings. Commands and health checks must include any required image environment setup. Services without profiles always run; a service with profiles runs only when one is selected. Start with repeated `--profile` options when several profiles are needed. If all services have profiles, select one explicitly; an empty selection does not enable everything.

## Dependencies and resources

`depends_on` supports `service_started` (default), `service_healthy`, or
`service_completed_successfully`. Dependencies must exist, be active in the
selected profiles, and form no cycle. Health checks establish readiness, not a
liveness kill loop. `restart: true` on an entry restarts the dependent after
that dependency restarts and meets the condition again, whether `antioch
service restart` or a scenario's `restart_services` restarted it; without it
the dependent keeps running.

One GPU class applies to the session; conflicting service classes are
refused. Engine services default to their engine's recommended class, and
non-engine projects without a GPU request use CPU capacity. Machine cells
expose GPUs to each application container; Kubernetes gives devices to one
service, preferring an engine service. Do not assume every helper can access CUDA; check the session's actual GPU allocation before depending on it. There are no fractional or cross-user allocations.

All of a session's services share one machine; `with:` placement groups are
accepted and ignored. A top-level `region: uk` (or `us-central`) pins every
session of the project there, including the ones Antioch starts for runs,
and `resources.region` still works as its spelling; a region no cell serves
is refused. Without a pin, placement prefers available compute, recent project reuse, and nearby capacity using measured latency before allocating more machines. Inspect the selected session's region rather than assuming every session lands in the same place.

Omitted CPU/memory means no request or limit; declared values reserve and cap
that service's resources (cores or milli-units, quantities such as `4Gi`).

Use the schema for `stop_grace_period`, `shm_size`, `tmpfs`, `user`, and
permitted `ipc`, `cap_add`, and `devices` values; some require machine cells
and are refused on Kubernetes cells. Restart defaults to `never`;
`on-failure` permits three retries, and `always` restarts after any exit.

## Routes

```yaml
ports:
  - name: viewer
    port: 8080
    protocol: tcp
    direction: client-to-service
```

Ports are named mappings, not Compose port strings. TCP is default; UDP is
supported. `client-to-service` exposes a service to the connected client;
`service-to-client` forwards to a registered client listener. Service names
resolve declared service routes; there is no public service IP. Names and
endpoints must be unique and platform routes are reserved. Use
[sessions](sessions.md) to bind and serve routes.

Isaac services show the native WebRTC stream in Mission Control. To show a
Viser app instead, declare its TCP `client-to-service` port and select it
with `viewer: {type: viser, port: PORT_NAME}` on the service. Use `viewer: {type: omniverse}` to explicitly choose the native stream without a port. Each service has one selected viewer, and Mission Control's **Viewer service** picker can follow active work or pin a service. Base compute has no implicit native video viewer.

A Viser service must start a Viser 1.1-or-later server on the declared port. Its scene and controls belong to that app; Antioch does not add simulation controls automatically. Access is restricted to the session owner. A five-minute viewer ticket refresh can reconnect the browser while the service keeps running. Rerun telemetry is separate from this interactive viewer.

For old manifests, move `x-antioch.viewer` onto the service and remove the retired `runner` field. History remains readable, but revisions using the removed extension need a new valid revision before running again.

## Source sync

```yaml
sync:
  - source: src
    target: /workspace/project/src
  - source: assets
    target: /workspace/project/assets
    exclude: ["*.tmp", "cache/**"]
```

A mapping copies one project-relative `source` (`.` for the project root)
onto an absolute `target` at or below `/workspace/project`; targets must not
overlap within a service. A source may start with `../` to reach code beside
the project, such as `source: ../robot_src`; it must exist where the command
runs. Files your `.gitignore` excludes are skipped even when a mapping names
them, reading ignore files as git does in the repository holding each source,
as are the built-in floor (VCS metadata, virtual environments, caches,
`outputs/`) and the mapping's own `exclude` patterns;
`.dockerignore` does not filter a mapping. Mapping exclusions do not support `!` negation. A symlink is copied as a link only when its target remains inside that mapping; other symlinks are skipped with a warning. If a remote folder conflicts with a local file, sync displaces it to `NAME.antioch-displaced-ID` instead of deleting the folder. Interrupted sync keeps completed transfers; the next command sends what remains.

Mappings copy files one way and never change images or revision identity;
[sessions](sessions.md) says when commands copy them. A service with no
mapping runs from its image alone, so put files a service command needs at
start in the image or under a mapping, and restart the service after editing
them. The generated `.dockerignore` is the reverse allowlist: `*` keeps synced
source out of the build, and a line such as `!requirements.txt` admits a file
the Dockerfile copies. Editing anything the build does not admit keeps the
image and the running session.

A schema 5 manifest's `watch` rules are read as mappings of the whole watched
path; their restart and exec actions are dropped with a coded notice. For
named scenario/case selections, use [suite YAML](suites.md).

Use [environment setup](environment.md) for build recipes, [sessions](sessions.md) for live sync and routes, and [scenario design](../../scenario-design/SKILL.md#inputs-and-execution-policy) for decorator service/profile/restart policy. [Antioch platform](../SKILL.md#capability-guides) links the other capabilities.
