# Machines and direct stack access

An Antioch machine is an ephemeral GPU VM and a user-isolation boundary. The
assignment gives the CLI an SSH capability for that VM; it is not a durable
checkout or source repository. Project services and direct connection commands
use the same assignment.

## Choose and inspect an assignment

```bash
antioch machine list --json
antioch machine checkout MACHINE --json
antioch machine status --machine MACHINE --json
```

Commands that drive one machine resolve it in this order: an explicit
`--machine MACHINE`, the project's checked-out machine, then the project's sole
assignment. If several assignments exist and none is checked out, pass the
selector explicitly. `machine list` defaults to the current project; add
`--all` to see every project's assignments. `machine checkout --none` clears
the local choice and
returns to the sole-assignment rule; a bare `antioch machine checkout` prints
the current machine id on stdout for scripts. `machine release --machine
MACHINE --json` stops assigned work and returns that VM to the pool; confirm
the target before running it. Release is idempotent — an already-released
machine reports success — so it is safe in teardown scripts. Queue-driven
assignments are refused with the platform's own message.

The text and JSON forms of `machine status` surface the machine's direct `url`
and, when available, its `stream_url`. Use those addresses when opening a
browser or diagnosing direct transport. Antioch authorizes the connection but
does not relay the simulation stream.

Machine list and status JSON use the curated user schema: identity, direct
URLs, GPU and placement, lifecycle state, generation, project assignment,
`dispatched_from` provenance, stream state, and the local `current`
selection. Provider instance and
boot-disk identities, organization internals, and machine certificates are
omitted;
all `*_at` values are Unix microseconds. See `cli.md` for the shared output
contract.

The machine list is personal. Scenario and suite history, assets, and usage
are shared with the organization, but another user's machine is not visible
to you.

## Track machine usage

```bash
antioch machine usage --json
antioch machine usage --since 30d --mine --json
```

`machine usage` reports measured GPU machine time over a window (`--since`
defaults to `7d`; `--until`, `--project`, `--user`, `--mine`, and `--limit`
narrow it). Machine time starts at assignment and ends at release: holding
two machines bills double, while several processes sharing one machine do not
multiply. Check usage before and after fan-out or long queued work, and
release machines the task no longer needs.

## Allocate a machine and start the project

Declare the stack in `antioch.yaml` and start it through the top-level CLI:

```bash
antioch services up --watch
```

`services up` allocates a machine when the project has no assignment, then
builds and
starts the selected services and waits for their health checks. `--watch` is
a foreground
development session and syncs declared rules while `ports` remain reachable.
Ctrl-C
ends that session but leaves containers and declared ports running;
`services down` stops them.
A bare `services up` also opens ports and returns after the services are
ready. The observation and teardown commands
`services ps`, `services logs`, and `services down` are resolve-only and never
allocate a machine. When no warm machine is available, the CLI itself waits
up to ten minutes for a replacement with named progress — do not wrap it in a
retry loop.

## Direct container and VM access

Name the service on `exec`; `ssh` alone defaults to `sim` when that service
exists:

```bash
antioch services exec sim nvidia-smi
antioch services exec sim bash -lc 'ls /workspace/output | wc -l'
antioch services ssh
```

Name an auxiliary service when it owns the command, for example
`antioch services exec ros bash -c "source /opt/ros/jazzy/setup.bash && ros2 topic list"` —
`exec` skips the image entrypoint, so source the environment a stock ROS
image prepares there. `services ssh` opens a human PTY in the selected
service; `antioch machine ssh` opens the VM shell. These connections resolve
an existing
assignment and do not allocate one. They are direct connections; use
`antioch run`, a scenario, or a suite when exit status,
results, telemetry, or artifacts need product truth.

Read service container logs directly, with tail and time bounds:

```bash
antioch services logs sim --tail 200
antioch services logs ros -f --since 10m
```

`services logs` streams raw container bytes to stdout. Note that
`antioch run` output does not land in the sim container log — it belongs to
the dispatched process (`scenario logs` replays a recorded run's output).

Use `antioch machine ssh` for the VM shell when a host-level diagnosis is the
explicit goal. Docker runs underneath the Antioch stack, so raw `docker ps`,
`docker logs`, and `docker exec` are supported there. The Compose vocabulary is
documented in [Docker's Compose file reference](https://docs.docker.com/reference/compose-file/).
Run one host command without opening an interactive shell when that is easier
to automate:

```bash
antioch machine ssh -- docker ps
```

The command runs without a PTY and returns the remote exit status.

Copy container paths with the top-level transfer verb:

```bash
antioch services cp sim:/workspace/output/ ./output/
antioch services cp sim:/workspace/output/result.png ./result.png --json
```

Name the service explicitly with `SERVICE:PATH` on exactly one side; use
`antioch machine ssh` for VM filesystem access.
For directory transfers, a destination ending in `/` is explicit directory
syntax and places the source basename below it. Without the trailing slash,
the destination path receives the source contents. Use the slash when a
script must make the destination directory boundary clear.
`/workspace/output` is assignment scratch and can disappear at release. Put
durable results in scenario artifacts or the asset store. Add `--json` for one
transfer manifest with the direction, path, byte count, checksum, and
replacement fact; symlinks are copied as links, never followed.

## Move a multi-GB dataset

Scenario artifacts and asset payloads accept up to 16 GiB per object, but the
tested dogfood paths were 64 MiB for an artifact and 128 MiB for `services cp`
and an asset. Treat those as confidence points, not a promise that one 10 GB
upload is a good retry unit. Antioch does not expose provider multipart uploads
or byte-range resume. A transfer is one object, so bound the failure domain by
sharding the dataset into archives such as `shard-0000.tar.zst` through
`shard-0015.tar.zst`.

Write one manifest beside the shards. Include the shard name, byte count, and
SHA-256 digest, for example:

```json
{
  "dataset": "warehouse-2026-08",
  "shards": [
    {"name": "shard-0000.tar.zst", "size_bytes": 734003200, "sha256": "..."}
  ]
}
```

For machine scratch, copy one shard at a time and keep the JSON transfer
manifest:

```bash
antioch services cp sim:/workspace/output/shard-0000.tar.zst ./shards/ --json
sha256sum ./shards/shard-0000.tar.zst
```

For durable scenario output, add each archive with a distinct artifact name and
download only the missing shard when a consumer retries:

```bash
antioch scenario download SCENARIO_RUN_ID --artifact shard-0000.tar.zst --output ./shards/shard-0000.tar.zst --json
```

For reusable organization data, publish each shard as its own immutable asset
version. A retry with the same name, version, and digest is idempotent; a
different digest is refused. Verify every pull against its recorded digest:

```bash
antioch assets pull datasets/warehouse-2026/shard-0000 --version v1 --output ./shards/shard-0000.tar.zst --json
antioch assets verify datasets/warehouse-2026/shard-0000 --version v1
```

If a shard fails, retry only that shard and update the manifest after its local
SHA-256 matches. Keep the manifest itself as a small scenario artifact or
versioned asset so the list of completed shards is durable.

## Keep a built image

A built service image lives on the machine and dies with it — released
generations cannot be reached by design. While the assignment is live, list
retained build products and export one to the local Docker daemon:

```bash
antioch services images --json
antioch services images pull sim --tag warehouse-sim
```

Reach for `services images pull` before releasing a machine whose build you
want to keep or inspect locally.

## Concurrency and streaming

Several processes, kernels, services, shells, and copies can share a machine.
The one scarce long-lived resource is its livestream. Simulation runs
(`antioch run`, scenario and suite dispatch) reserve it by default;
`antioch jupyter stream` claims it for a selected kernel. `services exec`
never touches it. An explicit `--stream` request is refused
when another process holds the lease; an unqualified default can continue
headlessly with a notice. See `running.md` and `sessions.md` for stream and
kernel workflows.
