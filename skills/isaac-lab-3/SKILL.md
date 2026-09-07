---
name: isaac-lab-3
version: "1.2.0"
description: Guides Isaac Lab environment, manager, controller, RL training, teleoperation, demonstration-dataset, and imitation-learning work in Antioch GPU sessions. Use for isaaclab imports or errors, manager-based and direct environments, observations/rewards/terminations, rsl-rl or skrl integration, Mimic, scaling, and migration from Lab 2.x. Covers lazy imports, XYZW quaternions, ProxyArray access, indexed/masked state writes, and backend selection. Not for plain Isaac Sim authoring (isaac-sim-6), Antioch dispatch/history (antioch-platform), or scenario verdict/telemetry design (scenario-design). Ground vendor API details with antioch-research.
---

# Isaac Lab on Antioch

This skill targets **Isaac Lab 3.0.0-beta2 on Isaac Sim 6.0.1**.
Load `antioch-platform` first. Use `antioch-research` to inspect the selected
runtime's API and source; not every indexed corpus has the same pin.
Beta interfaces can change between releases.

Preserve scope: explaining a task does not authorize training, and diagnosing
a run does not authorize changing its reward or termination contract.
For requested training, establish the workload and budget before dispatching
a long sweep.

## Startup and time

Antioch starts Lab through `AppLauncher`. A scenario runner owns startup;
the body must not create another launcher. Scripts and notebooks call
`antioch.start_simulation()` before simulator imports.

Keep `isaaclab*`, `isaacsim`, `omni`, `pxr`, and `carb` imports inside
functions or `TYPE_CHECKING`. Define config classes inside a factory called
after startup. A helper module is not exempt from import safety.

Use Lab's `SimulationContext` or environment loop, not `antioch.world()`
(which is Isaac Sim-only). Configure Lab timing with `SimulationCfg.dt`,
render interval, and environment decimation. Antioch's
`SimulationConfig.physics_dt/render_dt` configure the classic World and do
not set a Lab environment's step period.

`antioch.stage()` exposes the current stage; `antioch.application()` exposes
the app when Antioch started it. Everything beyond these handles is native
Lab. Do not mix the environment's stepping loop with an independent World.

## Migration boundaries

| Surface | Lab 3 contract |
|---|---|
| Quaternions | XYZW; identity `(0, 0, 0, 1)` |
| Array-valued state | `ProxyArray` access through `.torch` or `.warp` |
| Selected state writes | `write_*_to_sim_index` or `write_*_to_sim_mask` |
| Camera | `Camera/CameraCfg`; tiled names remain deprecated aliases |
| Physics config | `SimulationCfg.physics` with a backend-specific config |
| Native launcher | Headless default; visualizer options belong to the native launcher |

Do not infer that every `.data.*` property is an array; inspect the selected
property's type. Convert WXYZ at an Isaac Sim boundary into Lab's XYZW once.
A literal `(1, 0, 0, 0)` can be an intentional rotation, so check its meaning
rather than rewriting every occurrence mechanically.

Old state-write methods can forward with a deprecation warning at this pin.
Prefer the explicit selection variant in new code; validate index/mask shape
and device. `SceneEntityCfg.joint_ids/body_ids` may be `slice(None)`, not a
list. Resolve the selection against the articulation's actual ordered names.

## Array ownership and updates

`ProxyArray.torch` is a cached zero-copy tensor view; `.warp` exposes the
underlying Warp array. Use the explicit accessor for tensor methods. Clone
when you need an independent historical sample. Re-access the data property
after a full simulation reset or buffer recreation, especially on Newton:
an old wrapper or tensor can still refer to replaced storage. A saved view
is not a permanent handle to current state. Do not assume every per-environment
episode reset recreates buffers; check the selected backend's reset contract.

In a manually owned scene loop, write controls, step, then call
`scene.update(sim.get_physics_dt())`. A zero timestep does not advance
timestamp-based caches and is not a reliable "refresh now" operation.
Do not add duplicate updates around a managed environment that already owns
this lifecycle.

Keep device transfers out of the hot loop unless the consumer needs CPU data.
Check batch axes, environment origins, reference frames, and joint order
before computing errors.

## Backends

Lab supports `isaaclab_physx` and `isaaclab_newton`. Select matching launch
and environment configuration before constructing the scene. Concrete
physics configuration comes from the backend package, for example
`isaaclab_physx.physics.PhysxCfg`.

The managed Antioch path uses Kit; upstream kit-less examples are a different
launch path. Newton-native capabilities do not automatically imply support
through every Lab asset, sensor, or solver integration. Choose by the task's
required features and measured behavior, not a universal environment-count
claim or a prediction about the ecosystem.

## Environments and training

Read `references/env-authoring.md` for environment structure, resets, manager
terms, a small recorded rollout, and demonstration/imitation-learning workflows.
Prefer a shipped task close to the
requested robot/control problem, then make deliberate changes.

Ground the installed RL library version and its corresponding runner config.
rsl-rl 5.x uses separate actor/critic configuration; do not copy an older
`policy/ActorCritic` block without following the pinned migration path.
The shipped task's agent config and upstream training script are the starting
points. Verify checkpoint load/save and wrapper contracts too.

Before scaling, verify:

- Observation shape/order, frame, normalization, and finite values.
- Action scaling, limits, decimation, and actuator interpretation.
- Reward components and units, without replacing the task with easier rewards.
- Terminations versus time-limit truncations.
- Complete reset of physical and task state.
- Evaluation on held-out cases, not just training return.
- Saved checkpoints and their metadata through run artifacts.

Random actions are a startup smoke test, not policy success. Zero completed
episodes is not a measured mean return of zero. A finite loss or a saved
checkpoint alone does not prove useful behavior.

Scale `num_envs` gradually after a small run works. Profile cameras,
observations, scene state, and optimizer memory separately. There is no fixed
"512 cameras per GPU" or environment-count budget that fits all tasks.
Skipping appearance payloads is inappropriate when observations need them.

Record seeds, image/asset pins, config, metrics, and checkpoints. Seeding is
necessary for reproducibility but does not guarantee bit-identical GPU runs.
Use platform history and artifact readback to explain observed results;
do not describe an unrun training plan as verified.
