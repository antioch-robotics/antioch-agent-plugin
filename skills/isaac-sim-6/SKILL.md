---
name: isaac-sim-6
version: "1.1.5"
description: >-
  Guides Isaac Sim work in Antioch GPU sessions: scene and USD authoring,
  physics, sensors, asset import, navigation, manipulation, rendering, and
  synthetic data. Use when writing, porting, reviewing, or debugging Isaac Sim
  code for scenarios, suites, scripts, or notebooks. Covers simulator startup,
  lazy imports, live-state readback, and task-specific validation; routes API
  detail to antioch-research. Not for Antioch dispatch and asset-catalog
  operations (antioch-platform), scenario verdict/telemetry design
  (scenario-design), or Isaac Lab environments (isaac-lab-3).
---

# Isaac Sim on Antioch

This skill targets Isaac Sim **6.0.1 / Kit 110.1.2**. Code is authored without
a local simulator and runs in the remote simulator service. Load
`antioch-platform` first for project setup, dispatch, sessions, and readback.

Preserve the user's task. An API question does not authorize a GPU run.
A diagnosis does not authorize changing the physical model. For requested
implementation, use a small representative probe before a costly sweep.

## Ground the API

Use `antioch-research` before relying on a vendor API or behavior claim.
Inspect the exact runtime pin, signature, return shape, units, prerequisites,
and backend support. Dated documentation and source corpora can differ from
the image. A harvested type proves an interface, not a passing physical run.

## Who starts Kit

- A decorated scenario run starts Kit before the scenario body. Declare
  simulation needs in `@antioch.scenario(config=...)`; do not start another app.
- A plain script or notebook owns startup. Call
  `antioch.start_simulation()` before simulator imports.
- Use one simulation/stepping owner per process. Do not add a second
  `SimulationApp` around a framework that already owns one.

```python
import antioch


def main() -> None:
    antioch.start_simulation()
    world = antioch.world()
    # Build the scene here, then reset, step, and read live state.
```

A second identical startup is a no-op; a conflicting config raises.
`antioch.world()` returns the native classic Isaac `World` singleton and is
Isaac Sim-only. `antioch.stage()` returns the current USD stage.
`antioch.application()` returns the app when Antioch started it; an externally
started app retains its own handle.

### SimulationConfig

| Field | Default | Contract |
|---|---|---|
| `log_level` | Engine default | `fatal/error/warning/info/verbose` |
| `render_quality` | Engine default | `performance/balanced/quality/ultra` |
| `viewport` | 1280×720 | Headless render size; a streamed process ignores it |
| `physics_dt` / `render_dt` | 1/60 each | Render period must be an integer multiple of physics period |
| `physics_engine` | `"physx"` | `"newton"` selects the experimental Newton integration |
| `extensions` | `()` | Extra extension IDs enabled before the first stage |
| `extra_args` | `()` | Native Kit arguments appended after Antioch defaults |
| `stream` | Inherited | Launch preference; explicit CLI flags win |
| `timeout_s` | Inherited | Launch timeout; explicit CLI timeout wins |

For example, opt into `("isaacsim.ros2.bridge",)` or
`("isaacsim.sensors.experimental.physics",)` when needed. Installed extensions
are not all enabled automatically. Put custom extension files in the service
image and use `extra_args` for an extension search path if required.

Attached and detached dispatch have different stream behavior. Use the
platform skill rather than restating the whole CLI contract here.

## Lazy imports

`pxr`, `omni`, `carb`, `isaacsim`, and `isaaclab*` imports belong inside
functions or under `if TYPE_CHECKING:`. This includes helper modules and class
definitions that would import them transitively. Every project module must
remain importable without a simulator.

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from pxr import Usd


def current_stage() -> "Usd.Stage":
    import antioch

    return antioch.stage()
```

Do not import CUDA/engine integrations before their required startup boundary.
If a library such as torch conflicts with Kit startup, inspect the pinned
launcher and import order; "torch before settle always hangs" is not a
universal rule.

## Current and retained APIs

Use the pinned export and migration source to map each symbol. The main
current surfaces include:

| Work | Entry point |
|---|---|
| Experimental simulation lifecycle | `isaacsim.core.simulation_manager` |
| Live prim and articulation views | `isaacsim.core.experimental.prims` |
| Stage/app/transform utilities | `isaacsim.core.experimental.utils` |
| RTX sensors | `isaacsim.sensors.experimental.rtx` |
| Physics sensors | `isaacsim.sensors.experimental.physics` |
| Asset-root discovery | `isaacsim.storage.native` |

Classic `isaacsim.core.api.World` remains shipped and is what
`antioch.world()` returns. Several classic APIs are deprecated but present;
others, including old `omni.isaac.*` paths, were removed. Do not generalize
that all old APIs remain importable, or that every non-experimental API is
gone. Port the specific interface in scope.

Newton's extension has auto-switch behavior. Select the backend through
`SimulationConfig` before startup and verify
`SimulationManager.get_active_physics_engine()` if a result depends on it.
Do not enable Newton and then assume PhysX is still active.

## Build, step, observe

Inside a scenario, build the scene, initialize/reset through the chosen
framework, apply runtime controls, and advance using that framework's step.
With `antioch.world()` this is normally `world.reset()` followed by
`world.step(render=True)` when rendered evidence is needed.

A physics scene must exist, but `/World/PhysicsScene` is not a required path.
Kit app updates can advance a playing timeline; their count does not establish
a chosen physics timestep. Do not mix app pumping, classic World stepping,
and another framework's independent step loop without understanding ownership.

Read physics-backed state during simulation. Experimental
`RigidPrim.get_world_poses()` returns batched Warp position/quaternion arrays.
Select the intended body explicitly when converting a `(1, 3)` position.

USD transform reads depend on physics writeback and cache freshness. They can
be stale, but are not universally frozen at the initial pose. Use the owning
backend's live state for physical verdicts. Check camera/render synchronization
separately.

Isaac Sim quaternion APIs commonly use WXYZ; scipy and Isaac Lab 3 use XYZW.
Convert once at each boundary and check the chosen API, including any named
quaternion-order option.

## Evidence and reporting

Define the task's pass conditions before running. Record measured state and
named checks, not just logs or attractive frames. Keep failure samples for
diagnosis. Do not weld a payload, teleport a robot, remove obstacles, or loosen
a threshold to make a physical test pass.

A source-verified API, a CPU typecheck, a simulated result, and a rendered
artifact prove different things. Report them separately. Do not describe a
recipe as live-verified unless the exact relevant runtime probe ran.

## Domain references

Load only the domain needed. Each reference owns its whole workflow; detailed
vendor API lookup belongs to research rather than a second layer of recipes.

| Domain | Reference | Use for |
|---|---|---|
| Physics | `references/physics.md` | Backends, collisions, drives, live state, deformables, reset and stability |
| USD | `references/usd.md` | Layers, composition, transforms, instancing and packaged assets |
| Assets | `references/assets.md` | URDF/MJCF import, articulation inspection, units, placement and materials |
| Sensors | `references/sensors.md` | Cameras/calibration, lidar/radar/acoustic, IMU/contact and sensor data |
| Navigation | `references/navigation.md` | Maps, footprints, route clearance, mobile control and ROS export |
| Manipulation | `references/manipulation.md` | IK, grasp mechanics, transport and placement checks |
| Rendering | `references/rendering.md` | Lighting, capture, image diagnosis, video and batch resource ownership |
| SDG | `references/sdg.md` | Replicator, randomization, MobilityGen record/replay and dataset validation |
| Diagnosis | `references/troubleshooting.md` | Failure isolation, evidence and bounded probes |
