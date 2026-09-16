---
name: isaac-lab-3
version: "1.3.1"
description: Guides Isaac Lab 3.0.0-beta2 environments and manager terms, controllers, RL training, teleoperation, Mimic demonstrations, imitation learning, backend migration, and scaling in Antioch GPU sessions. Applies to `isaaclab*` imports or errors, environment observations/actions/rewards/terminations, rsl-rl or skrl integration, reset and state-write bugs, and Lab 2.x ports. Covers lazy imports, XYZW quaternions, ProxyArray access, indexed/masked writes, and PhysX/Newton selection; plain Isaac Sim authoring, Antioch dispatch/history, and scenario verdicts belong elsewhere, with vendor APIs grounded by antioch-research.
---

# Isaac Lab on Antioch

The pin is **Isaac Lab 3.0.0-beta2 on Isaac Sim 6.0.1**. Load
[Antioch platform](../antioch-platform/SKILL.md) for project/session setup and
[research](../antioch-research/SKILL.md) for methods, examples, and the
selected runtime's signatures and source; beta APIs can change.

## Startup and stepping

The scenario runner starts Lab through `AppLauncher`. A plain script or
notebook calls `antioch.start_simulation()` before simulator imports. Keep
`isaaclab*`, `isaacsim`, `omni`, `pxr`, and `carb` imports inside functions or
`TYPE_CHECKING`; define config classes in a post-startup factory so discovery
works without a simulator.

Use Lab's `SimulationContext` or environment loop, not `antioch.world()`.
Configure `SimulationCfg.dt`, render interval, and environment decimation;
classic `SimulationConfig.physics_dt/render_dt` does not set Lab timing. Keep
one stepping owner and do not mix an independent World loop.

## Lab contracts

| Surface | Contract |
|---|---|
| Quaternions | XYZW; identity is `(0, 0, 0, 1)`. |
| Array state | `ProxyArray`; access tensor methods through `.torch` or `.warp`. |
| Selected writes | `write_*_to_sim_index` or `write_*_to_sim_mask`; validate shape/device. |
| Cameras | `Camera`/`CameraCfg`; tiled names are deprecated aliases. |
| Physics | `SimulationCfg.physics` contains backend-specific configuration. |
| Launcher | Headless by default; visualizer options belong to the native launcher. |

Inspect each `.data.*` property's type; not every field is an array. Convert
WXYZ only at an Isaac Sim boundary. `SceneEntityCfg.joint_ids/body_ids` may be
`slice(None)`; resolve selectors against the articulation's ordered names.
Older state-write methods may forward with a deprecation warning at this pin;
prefer the explicit index/mask forms.

`ProxyArray.torch` is a cached zero-copy view and `.warp` exposes the Warp
array. Clone a historical sample. Re-access the data property after a full
reset or buffer recreation, especially on Newton; an old wrapper may refer to
replaced storage. An episode reset does not necessarily recreate buffers.

In a manually owned scene loop, write controls, step, then call
`scene.update(sim.get_physics_dt())`. A zero timestep is not a refresh. Do not
add duplicate updates around a managed environment. Keep device transfers out
of the hot loop unless the consumer needs CPU data, and check batch axes,
environment origins, frames, and joint order before computing errors.

## Backends and training

Lab supports `isaaclab_physx` and `isaaclab_newton`. Select matching launch and
environment configuration before constructing the scene; for example,
PhysX configuration comes from `isaaclab_physx.physics.PhysxCfg`. Newton-native
features do not imply support through every Lab asset, sensor, or solver.
The managed Antioch path runs Kit; kit-less upstream examples are a different
launch path. Choose the backend by required features and measured behavior.
Research custom Warp kernels and Newton integrations across libraries, then
verify compatibility with Lab's array and stepping contracts above. Select
images and dependencies through the platform's
[environment guide](../antioch-platform/references/environment.md).

Read [environment authoring](references/env-authoring.md) for manager/direct
structure, reset semantics, rollout probes, registration, RL adapters, and
demonstrations. Ground the installed RL library and runner config. rsl-rl 5.x
uses separate actor/critic configuration; follow the pinned task migration
instead of copying an older block. Verify checkpoint loading, wrapper reset
semantics, and a fresh evaluation run.

Before scaling, establish observation/action shapes and frames, normalization,
limits, decimation, actuator mode, reward terms/units, terminations versus
time-limit truncation, and complete reset of robot/task state. Random actions
are a startup smoke test, not policy success; a finite loss or checkpoint is
not useful behavior. Scale `num_envs` gradually and profile cameras, scene
state, observations, and optimizer memory separately.

Record seed, config, asset/image pins, metrics, and checkpoints. Stubs prove
interfaces, not physics, rendering, timing, or backend compatibility. Seeding
does not guarantee bit-identical GPU runs; report unrun training or evaluation
as unverified and use run artifacts for the evidence.
See [agentic simulation](../agentic-simulation/SKILL.md) for iterative experiments,
[scenario design](../scenario-design/SKILL.md) for recorded evaluations, and
[assets](../antioch-platform/references/assets.md) for reusable robots and worlds.
