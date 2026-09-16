---
name: isaac-sim-6
version: "1.3.2"
description: >-
  Guides Isaac Sim 6.0.1 scene, USD, physics, sensor, asset, navigation,
  manipulation, rendering, and synthetic-data work in Antioch GPU sessions.
  Applies when writing, porting, reviewing, or debugging Isaac Sim code in a
  scenario, suite, script, or notebook. Covers the remote Kit lifecycle, lazy
  imports, current experimental APIs, live-state readback, and evidence
  checks; dispatch and catalog work belong to antioch-platform, verdicts and
  telemetry to scenario-design, and Lab code to isaac-lab-3.
---

# Isaac Sim on Antioch

The runtime pin is **Isaac Sim 6.0.1 / Kit 110.1.2**. Author code without a
local simulator; Antioch runs it in the remote simulator service. Load
[Antioch platform](../antioch-platform/SKILL.md) for project and session setup;
its [environment guide](../antioch-platform/references/environment.md) owns
engine selection and dependencies.

The domain references preserve NVIDIA's specialist guidance and examples,
adapted at the startup, import, and execution boundaries below. Each names its
upstream source. Native APIs remain native; upstream source links do not imply
that a helper script or local installation is present in the project.

## Start Kit and own the loop

An Antioch scenario starts Kit before its body. A plain script or notebook
calls `antioch.start_simulation()` before simulator imports. One process has
one simulation and stepping owner; do not wrap a framework that already owns
an app in another `SimulationApp`.

```python
import antioch


def main() -> None:
    antioch.start_simulation()
    world = antioch.world()
    world.reset()
    world.step(render=True)
```

`antioch.world()` is the classic Isaac Sim `World` singleton; it is not an
Isaac Lab handle. `antioch.stage()` returns the current USD stage and
`antioch.application()` returns Antioch's app handle when Antioch started it.
Repeating an identical startup is a no-op; a conflicting startup config
raises.

`SimulationConfig` selects startup behavior:

| Field | Contract |
|---|---|
| `log_level` | `fatal`, `error`, `warning`, `info`, or `verbose`; default is the engine default. |
| `physics_dt`, `render_dt` | Render period is an integer multiple of physics period. |
| `physics_engine` | `physx` by default; `newton` selects Isaac's experimental Newton integration. |
| `extensions` | Extension IDs enabled before the first stage; installed does not mean enabled. |
| `renderer_quality` | `performance`, `balanced`, or `quality`; the global picture/RTX preset, not a sensor resolution. |
| `extra_args` | Native Kit arguments appended after Antioch defaults. |
| `stream`, `timeout_s` | Authored defaults; explicit CLI flags win. |

Enable optional robot, sensor, and bridge APIs through their owning extension
IDs in `extensions`; a Python import path is not always the extension ID.
For a missing module after startup, check extension availability and startup
errors before changing dependencies. Put custom extension files in the image.
Attached scenario/suite commands inherit stream and
timeout defaults; mixed defaults need explicit CLI values. See
[session modes](../antioch-platform/references/sessions.md) for dispatch. Native
startup honors authored `stream`, while `service exec` remains bounded by its
CLI `--timeout` (900 seconds by default).

## Import and API boundaries

Keep `pxr`, `omni`, `carb`, `isaacsim`, and `isaaclab*` imports inside a
function or under `if TYPE_CHECKING:`. This applies to helper modules and
transitive imports, so every project module imports without a simulator.

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from pxr import Usd


def stage() -> "Usd.Stage":
    import antioch

    return antioch.stage()
```

Use [research](../antioch-research/SKILL.md) for native examples and the exact
pinned signature, return shape, units, prerequisites, and backend support.
Harvested stubs establish an interface, not physics, rendering, timing, or
backend behavior.

Current surfaces include `isaacsim.core.simulation_manager`,
`isaacsim.core.experimental.prims`,
`isaacsim.core.experimental.utils`,
`isaacsim.sensors.experimental.rtx`,
`isaacsim.sensors.experimental.physics`, and
`isaacsim.storage.native`. Classic `isaacsim.core.api.World` remains shipped;
some classic APIs are deprecated and old `omni.isaac.*` paths were removed.
Check the symbol in scope instead of assuming either all legacy or all
non-experimental APIs are available.

Newton can auto-switch at startup. Select it through `SimulationConfig` before
the stage opens and verify
`SimulationManager.get_active_physics_engine()` when backend identity matters.
The standalone `newton` solver and Isaac's Newton USD/tensor integration are
different surfaces.

Custom Warp kernels and solver code can work alongside native APIs. Research
the required integration, device/array ownership, and stepping contract before
connecting them; a standalone solver's features need not exist through Isaac's
backend. See [physics](references/physics-simulation.md) for native setup and
[Isaac Lab](../isaac-lab-3/SKILL.md) for managed environments.

## Build, reset, step, and read live state

Build the scene, initialize views/controllers through the selected framework,
reset, apply controls, then step with that framework. With the classic World,
`World.step(render=True)` advances `render_dt / physics_dt` physics steps and
`render=False` advances one. Use `run.sim_s` or engine callback time, not a
step-call counter. App updates can advance a playing timeline but do not define
the physics timestep. A physics scene is required, but `/World/PhysicsScene` is
only a convention.

Experimental `RigidPrim.get_world_poses()` returns batched Warp position and
quaternion arrays; select the body explicitly, even for a `(1, 3)` position.
Read physics-backed state for physical checks. USD transforms and
`UsdGeom.XformCache` can lag physics writeback or cache edits, so they are not
an independent live-state query.

Isaac Sim commonly uses WXYZ quaternions; Isaac Lab 3 and scipy use XYZW.
Convert once at the boundary and confirm any API option that names an order.

## Evidence

Define the task oracle before running. Measure the requested behavior, retain
failed samples, and distinguish source-verified API, typecheck, runtime probe,
and rendered artifact. Proximity, a command acknowledgement, or a final
image is not physical contact or task success; use the domain reference that
owns the measurement. Report unrun runtime behavior as unverified.

## Domain references

Read the reference for the task; linked examples provide depth without loading
the whole library. Start with [assets](../antioch-platform/references/assets.md)
to find existing content in Antioch's catalog or Isaac's native library.

| Task | Native guidance |
|---|---|
| Bodies, collision, materials, joints, backends | [Physics](references/physics-simulation.md) |
| Layers, composition, variants, instancing | [USD architecture](references/usd-composition-architecture.md) |
| Asset packaging and delivery | [USD pipeline](references/usd-pipeline.md) |
| Bounds, units, placement, frames | [Spatial reasoning](references/spatial-reasoning.md) |
| Import URDF/MJCF and inspect robots | [Conversion](references/urdf-mjcf-to-usd-conversion.md), [articulations](references/usd-articulation.md) |
| Cameras, calibration, RTX and physics sensors | [Sensors](references/isaac-sim-sensor.md), [cameras](references/isaac-camera.md) |
| Maps, footprints, paths, wheeled control | [Navigation primitives](references/navigation-primitives.md), [occupancy maps](references/occupancy-map.md), [robot navigation](references/isaac-sim-robot-navigation.md) |
| Reach, grasp, transport, avoid obstacles | [Manipulation](references/manipulation-ik.md), [motion generation](references/motion-generation.md) |
| Lighting, materials, images, video | [Rendering](references/isaac-sim-rendering.md) |
| Annotated or mobile-robot datasets | [Data collection](references/data-collection-sim.md), [MobilityGen](references/mobility-gen.md) |
| Runtime failures or evidence review | [Troubleshooting](references/isaac-sim-troubleshooting.md), [validation](references/isaac-sim-validator.md) |

For a failure, start with the run's status, first error, input, and smallest
reproduction; use a bounded probe and keep the evidence with the result.
See [agentic simulation](../agentic-simulation/SKILL.md) to drive that loop and
[scenario design](../scenario-design/SKILL.md) for recorded checks and telemetry.
