# The project manifest

`antioch.yaml` is a closed, Compose-inspired schema for identity, services,
scenario discovery, and suites. Start from `antioch init` and validate with
the installed `ProjectManifest` model. Unknown fields are rejected.

## Services and images

```yaml
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
    watch:
      - action: sync
        path: .
        target: /workspace/project
```

Preserve an existing ID. At least one service is required, each with exactly
one `image` or `build`. The engine image identifies the simulator role;
the service name does not. With multiple engine-backed services, set
`x-antioch: {runner: true}` on the intended runner.

Supporting services can declare commands, environment, working directory,
profiles, dependencies, health checks, resources, restart policy, and routes.
Commands and health checks must include any required image environment setup.

## Dependencies and resources

`depends_on` supports `service_started` (default), `service_healthy`,
or `service_completed_successfully`. Dependencies must exist, be active in
the selected profiles, and form no cycle. Health checks establish startup and
readiness, not a liveness kill loop.

One GPU class applies to the session; conflicting service classes are refused.
Engine services default to their engine's recommended class. Non-engine
projects without a GPU request use CPU capacity. All colocated services see
the machine's GPUs. There are no fractional or cross-user GPU allocations.

`with: all` (the default) colocates services; `with: SERVICE` joins that
service's placement group; `with: none` separates it. This release supports
one placement group per session and refuses multi-node placement.

Omitted CPU/memory means no request or limit. Declared values reserve and cap
that service's resources. CPU accepts cores or milli-units; memory accepts
quantities such as `4Gi`. `antioch project show --json` shows placement
and resolved requests.

Use the schema for `stop_grace_period`, `shm_size`, `tmpfs`, `user`,
and permitted `ipc`, `cap_add`, and `devices` values; some require machine
cells and are refused on Kubernetes cells. Restart defaults to `never`;
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
supported. `client-to-service` exposes a service to the connected client.
`service-to-client` forwards to a registered client listener. Service names
resolve declared service routes; there is no public service IP to manage.
Names/endpoints must be unique and platform routes are reserved.
Use [sessions](sessions.md) to bind and serve routes.

## Development watch

```yaml
watch:
  - action: sync+exec
    path: schemas
    target: /workspace/project/schemas
    command: ["python", "-m", "tools.compile_schemas"]
    timeout: 30s
```

Watch paths are project-relative; targets must stay under
`/workspace/project` and cannot overlap within a service. Rules can use
`include` and `ignore` patterns.

- `sync` copies files.
- `sync+restart` copies and restarts the service process in its existing
  container, waiting for fresh health.
- `sync+exec` copies and runs the command with a positive timeout of at most
  five minutes.

Watch never changes images or revision identity. Background runs do not apply
watch rules; their Dockerfile must include required source.
For named scenario/case selections, use [suite YAML](suites.md).
