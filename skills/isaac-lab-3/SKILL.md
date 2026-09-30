---
name: isaac-lab-3
version: "1.3.4"
description: Use for Isaac Lab environments, manager terms, controllers, RL training, teleoperation, demonstrations, Mimic, imitation learning, resets, backend migration, and scaling on Antioch. Covers XYZW quaternions, ProxyArray, indexed writes, and PhysX/Newton selection; use Isaac Sim for plain scenes and scenario-design for recorded verdicts.
---

# Isaac Lab on Antioch

The pin is **Isaac Lab 3.0.0-beta2 on Isaac Sim 6.0.1**. Load
[Antioch platform](../antioch-platform/SKILL.md) for project/session setup and
[research](../antioch-research/SKILL.md) for the selected runtime's
signatures and source; beta APIs can change.

## Startup and stepping

The scenario runner starts Lab through `AppLauncher`. A plain script or
notebook calls `antioch.start_simulation()` before simulator imports. Keep
`isaaclab*`, `isaacsim`, `omni`, `pxr`, and `carb` imports inside functions
or `TYPE_CHECKING`; define config classes in a post-startup factory so
discovery works without a simulator.

Use Lab's `SimulationContext` or environment loop, not `antioch.world()`.
Configure `SimulationCfg.dt`, render interval, and environment decimation;
classic `SimulationConfig.physics_dt/render_dt` does not set Lab timing. Keep
one stepping owner.

## Lab contracts

| Surface | Contract |
|---|---|
| Quaternions | XYZW; identity is `(0, 0, 0, 1)`. |
| Array state | `ProxyArray`; access tensor methods through `.torch` or `.warp`. |
| Selected writes | `write_*_to_sim_index` or `write_*_to_sim_mask`; validate shape/device. |
| Cameras | `Camera`/`CameraCfg`; tiled names are deprecated aliases. |
| Physics | `SimulationCfg.physics` contains backend-specific configuration. |
| Launcher | Headless by default; visualizer options belong to the native launcher. |

Check each field's type before accessing array methods. Clone historical
samples and refresh views after a reset that replaces storage. In a manual
scene loop, write controls, step, then `scene.update(sim.get_physics_dt())`;
a managed environment owns those updates.

## Backends and training

Lab supports `isaaclab_physx` and `isaaclab_newton`. Select matching launch
and environment configuration before constructing the scene; PhysX
configuration comes from `isaaclab_physx.physics.PhysxCfg`. Newton-native
features do not imply support through every Lab asset, sensor, or solver,
and the managed Antioch path runs Kit; kit-less upstream examples are a
different launch path. Select images and dependencies through the platform's
[environment guide](../antioch-platform/references/environment.md).

Load [environment authoring](references/env-authoring.md) for state/reset
contracts, a rollout probe, RL adapters, and demonstration workflows. Before
scaling, verify observations, actions, frames, limits, termination conditions,
and complete resets. Random actions test startup, not policy quality; finite
loss or a checkpoint does not prove useful behavior.

Save seeds, configuration, asset/image pins, metrics, and checkpoints. GPU
seeding does not guarantee identical runs. Use [scenario design](../scenario-design/SKILL.md)
for recorded evaluations and [agentic simulation](../agentic-simulation/SKILL.md)
for iterative experiments.

## Load the part you need

| Task | Guide |
|---|---|
| Define manager terms, actions, observations, rewards, or episode resets | [Environment structure and state](references/env-authoring.md#choose-the-environment-boundary) |
| Check a shipped task before adding complexity | [Small rollout probe](references/env-authoring.md#small-startup-probe) |
| Register a task, choose an RL adapter, or load a checkpoint | [Registration and RL](references/env-authoring.md#registration-and-rl) |
| Record/replay demonstrations or generate Mimic data | [Demonstrations and imitation](references/env-authoring.md#demonstrations-and-imitation) |
| Scale environments while keeping a useful evaluation | [Scaling and evaluation](references/env-authoring.md#scaling-and-evaluation) |
| Import a robot or validate its articulation | [Conversion](../isaac-sim-6/references/urdf-mjcf-to-usd-conversion.md) and [articulations](../isaac-sim-6/references/usd-articulation.md) |

These guides adapt native workflow ownership to Antioch sessions. They do not replace Lab's native scene, manager, training, or reset APIs. Use the pinned upstream implementation through [research](../antioch-research/SKILL.md) when selecting an unfamiliar term or backend feature.
