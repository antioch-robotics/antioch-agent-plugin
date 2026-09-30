---
name: isaac-sim-6
version: "1.4.4"
description: Use for Isaac Sim scene, USD, physics, robot, sensor, rendering, and synthetic-data code in Antioch. Covers startup and native API boundaries; load the task-specific reference for worked examples and depth. Use isaac-lab-3 for Lab environments and scenario-design for evaluations.
---

# Isaac Sim on Antioch

The runtime pin is **Isaac Sim 6.0.1 / Kit 110.1.2**. Author code without a
local simulator; Antioch runs it in the remote simulator service. Load
[Antioch platform](../antioch-platform/SKILL.md) for project and session
setup; its [environment guide](../antioch-platform/references/environment.md)
owns engine selection and dependencies.

The domain references adapt NVIDIA's specialist guidance at the startup,
import, and execution boundaries below; each names its upstream source.
Upstream script links are source examples, not helpers installed in the project. Retain their native workflows and examples when adapting code. Change startup, imports, and execution for Antioch; check APIs against the runtime pin and correct specific source errors rather than dropping the surrounding explanation.

Start from the task table below. Each reference links to its prerequisites, related tasks, and upstream source. Follow those links as the task moves from asset preparation to control, sensors, and evaluation; you do not need to load the whole library.

## Startup and imports

Scenarios start Kit before their body. A script or notebook calls
`antioch.start_simulation()` first, then imports simulator modules inside
functions. Keep `pxr`, `omni`, `carb`, `isaacsim`, and `isaaclab*` out of
module-scope imports, including helper modules; `TYPE_CHECKING` is safe.
One process has one app and stepping owner. Do not create another
`SimulationApp` around a framework that owns one.

`antioch.world()` returns the classic Isaac Sim World, not an Isaac Lab
handle; `antioch.stage()` returns the USD stage, and `antioch.application()` returns the app handle. Identical startup is a no-op;
a conflicting configuration needs a new process. For a complete script and
`SimulationConfig` options, read [simulation code](../antioch-platform/references/simulation-code.md).

For types without importing the simulator during local discovery:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from pxr import Usd


def current_stage() -> "Usd.Stage":
    import antioch

    return antioch.stage()
```

`SimulationConfig.viewport_updates` controls the default viewport independently of streaming. `None` follows the stream; `True` keeps viewport rendering enabled, and `False` pauses it, including any stream image. Use `True` for code that waits on viewport rendering, such as a Replicator orchestrator running without a stream. Kit must still receive updates from the stepping owner. See [rendering](references/isaac-sim-rendering.md) and [simulation configuration](../antioch-platform/references/simulation-code.md).

## Native APIs and live state

Use [research](../antioch-research/SKILL.md) for pinned signatures, units,
prerequisites, and backend support. Current surfaces include
`isaacsim.core.experimental.prims`, `isaacsim.core.experimental.utils`,
`isaacsim.core.simulation_manager`, `isaacsim.sensors.experimental.rtx`,
`isaacsim.sensors.experimental.physics`, and `isaacsim.storage.native`; classic `isaacsim.core.api.World` still
ships. Do not copy retired `omni.isaac.*` imports.

Build, initialize, reset, control, and step with the selected framework.
Classic `World.step(render=True)` advances `render_dt / physics_dt` physics
steps; `render=False` advances one. Measure progress with simulation time,
not the number of render calls. Use `run.sim_s` in a scenario or the engine clock/callback time in a script. Read physics-backed poses for physical
checks; authored USD transforms can lag writeback. Experimental
`RigidPrim.get_world_poses()` returns batched Warp arrays, even for one body.

Isaac Sim commonly uses WXYZ quaternions; Lab 3 and scipy use XYZW. Convert
at the boundary. Select a backend before stage construction, and verify
coupled behavior: standalone Newton support does not prove support through
Isaac's USD/tensor integration. Verify the active backend with `SimulationManager.get_active_physics_engine()` when it affects the result. Warp arrays and kernels can be part of the native implementation; check device, dtype, batch shape, ownership, and synchronization when exchanging them with NumPy, Torch, or physics tensors.

## Evidence

Define the success check before running. Source inspection verifies an API contract; a runtime probe verifies only the behavior it exercises. A command acknowledgement, nearby objects, or a final picture does not prove physical contact or task completion. Keep live measurements, run inputs, checks, and artifacts together through [scenario design](../scenario-design/SKILL.md), and distinguish simulation results from real-world validation.

## Domain references

Start with [assets](../antioch-platform/references/assets.md) to find
existing content in Antioch's catalog or Isaac's native library.

| Task | Native guidance |
|---|---|
| Bodies, collision, materials, joints, backends | [Physics](references/physics-simulation.md) |
| Layers, composition, variants, instancing | [USD architecture](references/usd-composition-architecture.md) |
| Asset packaging and delivery | [USD pipeline](references/usd-pipeline.md) |
| Bounds, units, placement, frames | [Spatial reasoning](references/spatial-reasoning.md) |
| Layout algorithms, containment, repeated assets, support surfaces | [Advanced spatial reasoning](references/spatial-reasoning-advanced.md) |
| Impact, feeders, spinning bodies, contact chains, mechanisms | [Physics experiments](references/physics-simulation-examples.md) |
| Import URDF/MJCF and inspect robots | [Conversion](references/urdf-mjcf-to-usd-conversion.md), [articulations](references/usd-articulation.md) |
| Cameras, calibration, RTX and physics sensors | [Sensors](references/isaac-sim-sensor.md), [cameras](references/isaac-camera.md) |
| Maps, footprints, paths, wheeled control | [Navigation primitives](references/navigation-primitives.md), [occupancy maps](references/occupancy-map.md), [robot navigation](references/isaac-sim-robot-navigation.md) |
| Reach, grasp, transport, avoid obstacles | [Manipulation](references/manipulation-ik.md), [motion generation](references/motion-generation.md) |
| Lighting, materials, images, video | [Rendering](references/isaac-sim-rendering.md) |
| Annotated or mobile-robot datasets | [Data collection](references/data-collection-sim.md), [MobilityGen](references/mobility-gen.md) |
| Runtime failures or evidence review | [Troubleshooting](references/isaac-sim-troubleshooting.md), [validation](references/isaac-sim-validator.md), [quality criteria](references/isaac-sim-validator-references-quality-criteria.md) |
| ROS 2 topics, TF, Nav2, or MoveIt 2 on a simulated robot | [ROS 2](../ros2/SKILL.md) |

For motion-generation work, load the specific step you are implementing: [workflow](references/motion-generation-references-workflow.md), [world binding](references/motion-generation-references-world-binding.md), [control loop](references/motion-generation-references-control-loop.md), [frames and grasping](references/motion-generation-references-frames-and-grasping.md), or [cuMotion setup](references/motion-generation-references-cumotion.md).

For a failure, start with the run's status, first error, input, and smallest
reproduction. See [agentic simulation](../agentic-simulation/SKILL.md) to
drive that loop and [scenario design](../scenario-design/SKILL.md) for
recorded checks and telemetry.
