# Machines and direct stack access

An Antioch machine is an ephemeral GPU VM and a user-isolation boundary. The
assignment gives the CLI an SSH capability for that VM; it is not a durable
checkout or source repository. Project services and direct connection commands use the
same assignment.

## Choose and inspect an assignment

```bash
antioch machine list --json
antioch machine checkout MACHINE --json
antioch machine status --machine MACHINE --json
```

Commands that drive one machine resolve it in this order: an explicit
`--machine MACHINE`, the project's checked-out machine, then the project's sole
assignment. If several assignments exist and none is checked out, pass the
selector explicitly. `machine checkout --none` clears the local choice and
returns to the sole-assignment rule. `machine release --machine MACHINE
--json` stops assigned work and returns that VM to the pool; confirm the target
before running it.

The text and JSON forms of `machine status` surface the machine's direct `url`
and, when available, its `stream_url`. Use those addresses when opening a
browser or diagnosing direct transport. Antioch authorizes the connection but
does not relay the simulation stream.

Machine list and status JSON use the curated user schema. Provider instance and
boot-disk identities, organization internals, and machine certificates are omitted;
all `*_at` values are Unix microseconds. See `json.md` for the shared output
contract and the command's `--help` for its current fields.

The machine list is personal. Scenario and suite history, assets, and usage are
shared with the organization, but another user's machine is not visible to you.

## Allocate a machine and start the project

Declare the stack in `antioch.yaml` and start it through the top-level CLI:

```bash
antioch services up --watch
```

`services up` allocates a machine when the project has no assignment, then builds and
starts the selected services and waits for their health checks. `--watch` is a foreground
development session and syncs declared rules while `ports` remain reachable. Ctrl-C
ends that session but leaves containers and declared ports running; `services down` stops them.
A bare `services up` also opens ports and returns after the services are ready. The observation and teardown commands
`services ps`, `services logs`, and `services down` are resolve-only and never allocate a machine.

## Direct container and VM access

The top-level connection verbs default to the required `services.sim` service:

```bash
antioch services exec sim nvidia-smi
antioch services exec sim bash -lc 'ls /workspace/output | wc -l'
antioch services ssh
```

Name an auxiliary service when it owns the command, for example
`antioch services exec ros bash -c "source /opt/ros/jazzy/setup.bash && ros2 topic list"` —
`exec` skips the image entrypoint, so source the environment a stock ROS
image prepares there. `services ssh` opens a human PTY in the selected
service; `antioch machine ssh` opens the VM shell. These connections resolve an existing
assignment and do not allocate one. They are direct connections; use
`antioch run`, a scenario, or a suite when exit status,
results, telemetry, or artifacts need product truth.

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
antioch services cp sim:/workspace/output/ local.txt
antioch services cp sim:/workspace/output/result.png ./result.png --json
```

Name the service explicitly with `SERVICE:PATH`; use `antioch machine ssh` for VM filesystem access.
`/workspace/output` is assignment scratch and can disappear at release. Put
durable results in scenario artifacts or the asset store. Add `--json` for one
transfer manifest with the direction, path, byte count, checksum, and
replacement fact. See `antioch services cp --help` for the exact path-direction
and service options.

## Concurrency and streaming

Several processes, kernels, services, shells, and copies can share a machine.
The one scarce long-lived resource is its livestream. Simulation runs reserve
it by default, direct `exec` uses it only with explicit `--stream`, and
`jupyter stream` claims it for a selected kernel. An explicit request is refused
when another process holds the lease; an unqualified default can continue
headlessly with a notice. See `running.md` and `sessions.md` for stream and
kernel workflows.
