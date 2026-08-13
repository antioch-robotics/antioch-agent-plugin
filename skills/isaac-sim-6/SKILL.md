---
name: isaac-sim-6
description: >
  Guides any Isaac Sim work that runs remotely on Antioch GPU machines:
  scene, physics, and USD authoring; sensors and cameras (RTX cameras, lidar,
  IMU, contact); asset import and articulation (URDF/MJCF to USD); mobile-robot
  navigation; arm manipulation and grasping; rendering, lighting, and capture;
  and synthetic data generation with Replicator and MobilityGen. Teaches the
  lazy-import invariant, the isaacsim.* namespace map, the boot/step/readback
  loop, and classic Kit failure modes. Use proactively for ANY Isaac Sim task —
  writing, porting, reviewing, or debugging Isaac code destined for Antioch
  scenarios, queued suites, or one-off scripts — then route API detail to the
  antioch-research tools and depth questions to domain references below. Not
  for dispatch, run history, or asset-catalog search (antioch-platform),
  scenario verdict and telemetry design (scenario-design), or Isaac Lab code
  (isaac-lab-3).
---

# Isaac Sim 6 for Antioch

Code written here is authored locally (no simulator installed) and executed
remotely on Antioch GPU machines running Isaac Sim 6.0.1 (which embeds Kit
110.1.2). How to dispatch, stream logs, and read back artifacts: load the
`antioch-platform` skill. This skill is only about writing correct Isaac code.

Isaac runs on the remote machine. Simulation commands stream its GUI to the
browser by default, and `--no-stream` runs headlessly. In either mode, verify
the scenario through recorded metrics, telemetry, and artifacts rather than
relying only on the viewport.

## Research first

Before writing or debugging ANY code that touches an Isaac Sim, Omniverse/Kit,
OpenUSD, PhysX, Newton, Warp, or Replicator surface, ground it with the
antioch-research MCP: `research_search` for signatures, parameters, defaults,
and behavior (`kind="source"` to localize the implementing code), and
`research_open` to read a whole vendor file. The corpora are pinned to the
exact versions on the machine (Isaac Sim 6.0.1 docs + source, plus Kit,
OpenUSD, PhysX, Newton, Warp, and more). This skill orients — concepts,
invariants, and recipes; research grounds — never trust a remembered API
signature over a retrieved one.

## Running on Antioch — who boots Kit

Every run is one process holding one Kit. Who starts it depends on the entry
point:

- **Scenarios** (`@antioch.scenario` + `antioch scenario run`): the runner
  boots Kit before calling the function. Scenario bodies never call
  `antioch.boot()` and never construct `SimulationApp` — declare timing needs
  on the decorator:
  `@antioch.scenario(sim=antioch.BootProfile(physics_dt=1 / 120, render_dt=1 / 60))`.
- **Plain scripts** (`antioch run script.py`) and **notebooks**: the code owns
  the lifecycle. Call `antioch.boot()` once, before any Isaac import:

```python
def main() -> None:
    antioch.boot()  # first — before importing pxr / omni / carb / isaacsim

    from isaacsim.core.experimental.prims import RigidPrim

    ...
```

`antioch.boot(...)` options (all keyword-only, all optional):

| Option | Default | Effect |
|---|---|---|
| `log_level` | Isaac defaults | Minimum Kit severity: `fatal`/`error`/`warning`/`info`/`verbose` |
| `render_quality` | Isaac defaults | DLSS preset: `performance`/`balanced`/`quality`/`ultra` |
| `viewport` | 1280×720 | Headless render size; a streamed boot ignores it |
| `physics_dt` / `render_dt` | 1/60 each | Physics substep and render step; `render_dt` must be an integer multiple of `physics_dt` |
| `extra_args` | `()` | Native Kit CLI arguments appended after Antioch's defaults |

Streaming is exclusively the CLI's, and simulation runs stream **by
default** — `antioch run`, `antioch scenario run`, and `antioch suite run`
reserve the machine livestream unless you pass `--no-stream`.
`antioch services exec` never reserves the stream (it has no stream option),
and `antioch jupyter stream` declares it for a kernel **before**
`antioch.boot()` is called in a cell or notebook. Neither
`antioch.boot()` nor `BootProfile` accepts a `stream` option or stream tuning;
encoder defaults belong to the transport.

A second identical call is a no-op; a conflicting one raises — Kit cannot
re-initialize in one process. In scenarios, streaming and log level are CLI
concerns (`antioch scenario run`, which streams by default), overlaid on the
decorator's profile.

Convenience handles into native state, safe any time after boot:

- `antioch.world()` — Isaac's native `World` singleton, created with the boot
  profile's `physics_dt`/`render_dt`. Isaac Sim engine only.
- `antioch.stage()` — the current USD stage, on any engine.
- `antioch.application()` — the live `SimulationApp` (only when Antioch booted
  it; notebooks with a self-booted app use `world()`/`stage()`).
- `antioch.is_running()` / `antioch.engine()` — liveness probe and the image's
  engine identity.

Everything past these handles is **native Isaac Sim** — Antioch adds no
wrapper layer over scenes, physics, sensors, USD, or Replicator. Write the
native APIs below exactly as NVIDIA documents them; the references in this
skill exist to get that code right.

## Rule #1 — lazy simulator imports (hard invariant)

`pxr`, `omni`, `carb`, `isaacsim`, and `isaaclab*` must NEVER be imported at
module scope. Antioch discovers scenarios locally with no simulator
installed — a module-scope simulator import breaks discovery and fails loudly
at `antioch.core.kit.boot()`.

```python
# WRONG — crashes local discovery
from pxr import UsdPhysics
import isaacsim.core.experimental.prims as prims


# RIGHT — imports inside the function body
def build_scene() -> None:
    from pxr import UsdPhysics
    from isaacsim.core.experimental.prims import RigidPrim

    ...
```

For type annotations use `if TYPE_CHECKING:`:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from pxr import Usd


def get_stage() -> "Usd.Stage":
    import antioch

    return antioch.stage()
```

A separate boot-ordering note, not part of the discovery invariant: `torch`
and `scipy` are importable at module scope without breaking discovery, but
deferring their import until after the app boots avoids CUDA-context hangs in
Kit. Keep them inside function bodies as well, for a different reason.

## Namespace map — Isaac Sim 6.0.1

Two removals hard-fail in 6.0: anything imported from `omni.isaac.*`, and
the `set_next_simulation_time` event API. Everything else old is deprecated
but still importable — port it, but don't treat it as dead:

| Removed | Replacement |
|---|---|
| `omni.isaac.*` (anything) | the matching `isaacsim.*` namespace below — no shim exists |
| `set_next_simulation_time` | multitick rendering (see stepping below) |

| Deprecated but present | Preferred API |
|---|---|
| `isaacsim.core.api` | `isaacsim.core.simulation_manager`, `isaacsim.core.experimental.prims` |
| `isaacsim.core.prims` | `isaacsim.core.experimental.prims` |
| `isaacsim.core.utils` | `isaacsim.core.experimental.utils` |
| `isaacsim.sensors.camera` | `isaacsim.sensors.experimental.rtx` |
| `isaacsim.sensors.physics` | `isaacsim.sensors.experimental.physics` |
| `isaacsim.robot_motion.lula` | `isaacsim.robot_motion.pink` (PINK IK) / `cumotion` |

`isaacsim.core.api` still exports `World`, `SimulationContext`, and
`PhysicsContext` in 6.0.1 — Antioch's own `antioch.world()` returns that
`World`, so the deprecated accessor remains the supported path for the
stepping loop below.

Both physics engines consume the same `UsdPhysics.*` schema; pick the backend
at runtime (inside a function, per Rule #1):

```python
def use_newton() -> None:
    from isaacsim.core.simulation_manager import SimulationManager

    SimulationManager.switch_physics_engine("newton")  # or "physx"
```

Newton is experimental in 6.0. Enabling the `isaacsim.physics.newton`
extension registers it but does not activate it by default — activation needs
`/exts/isaacsim.physics.newton/auto_switch_on_startup=true` or an explicit
`switch_physics_engine("newton")`. When it matters, check
`SimulationManager.get_active_physics_engine()` rather than assuming PhysX.

## The boot / step / readback loop

These ordering rules fix most "my sim does nothing" bugs:

1. Boot first. Nothing `pxr`/`omni`/`carb`/`isaacsim`/`isaaclab*` may be
   imported before the SimulationApp boundary (`antioch.boot()` / scenario
   boot). CUDA and Kit extension state must exist first.
2. Define `/World/PhysicsScene` before playing the timeline, or the first
   `play()` can hang.
3. `play(commit=True)` before any sensor `get_data()`. Experimental sensors
   return empty/stale data otherwise. Do not call legacy `World.initialize()`.
4. Set velocities after `timeline.play()` — the authored USD
   `physics:velocity` attribute is ignored by PhysX at runtime. Use
   `RigidPrim.set_velocities(linear_velocities=...)` post-play.
5. Advance through `World.step(render=True)` (or the
   `SimulationContext.step(render=True)` singleton). `simulation_app.update()`
   alone does not sync physics.

Inside a scenario the runner has already booted Kit, so the loop starts at
`antioch.world()`:

```python
@antioch.scenario()
def settle_check(run: antioch.ScenarioRun) -> None:
    from isaacsim.core.experimental.prims import RigidPrim

    world = antioch.world()
    die = RigidPrim(paths="/World/Die")
    world.reset()  # plays the timeline and commits initial physics state
    die.set_velocities(linear_velocities=...)  # after play, not before
    for _ in range(240):  # several seconds of physics before judging stability
        world.step(render=True)
    pos, quat = die.get_world_poses()  # warp arrays; .numpy() to convert
```

The same body works in an `antioch run` script or notebook with one
`antioch.boot()` call first — see "Running on Antioch" above.

## Reading state — the XformCache trap

The single most common silent bug: reading the authored USD transform instead
of the simulated state.

| Source | Returns | Use when |
|---|---|---|
| `UsdGeom.XformCache.GetLocalToWorldTransform()` | authored layer pose | pre-play editing only |
| `RigidPrim.get_world_poses()` | simulated state | always during sim |
| `Articulation.get_world_poses()` | simulated state | articulated robots |

Physics writes to Fabric, not the USD layer — `XformCache` returns the initial
pose forever once the sim is running. Depth: `references/physics.md`,
"Physics-to-USD readback".

## Quaternions

USD / Isaac Sim uses `[w, x, y, z]`; scipy uses `[x, y, z, w]`. Convert at the
boundary, never mix:

```python
from scipy.spatial.transform import Rotation

r = Rotation.from_quat([q[1], q[2], q[3], q[0]])  # WXYZ -> scipy XYZW
```

(Isaac Lab 3.x flips to XYZW globally — see the `isaac-lab-3` skill. Do not
carry one convention into the other stack.)

## Remote-first working style

- The stage runs on the machine, not on the laptop. Use the browser stream for
  interactive inspection, then assert on simulated state numerically (poses,
  contact forces, joint positions) and save the measurements with the run.
- Before importing or converting a robot, prop, or environment the user
  names, search the shared asset catalog first
  (`antioch assets list -q franka --json` — the `antioch-platform` skill's
  assets reference); import only what the catalog does not already hold.
- Prefer a decorated scenario over `antioch run` the moment evidence must
  persist — metrics, artifacts, and telemetry survive only on a recorded run.
- First-launch MDL shader compile takes 5–15 min on a cold machine — a slow
  first step is not a hang. Antioch's warm images pre-warm this, but budget
  for it on cold starts.
- Replicator capture needs the timeline playing and one render step before
  `get_data()`; verify captures by reading the artifact back, not by eye. A
  beauty-render warm-up loop is different: `update_app()` advances RTX's
  denoiser and temporal accumulation. Start with roughly 100–200 frames for an
  ordinary shot and 300–500 for deeply occluded/path-traced interiors, then
  stop when a frame-statistics probe is stable for several frames — those are
  budgets, not magic constants.
- Sim duration matters: give a scene a few seconds of physics (on the order
  of ~3 s in NVIDIA's stability benchmarks) before judging whether it has
  settled — re-measure in your own scene rather than trusting one number.

## Common gotchas

1. `omni.isaac.*` anywhere → hard import failure in 6.0. Rewrite to the map above.
2. Module-scope `pxr`/`omni`/`carb`/`isaacsim`/`isaaclab*` import → breaks
   Antioch scenario discovery.
3. `RigidBodyAPI` and `CollisionAPI`: for a simple body keep both on the same
   prim; for a compound body put the colliders on child prims of the prim that
   carries `RigidBodyAPI` (USD assigns each collider to the nearest ancestor
   with a rigid-body API). See `references/physics.md`.
4. `Cube.size=1.0` means half-extent 0.5, not 1.0.
5. `physics:angularVelocity` USD attr is in degrees/s, not rad/s.
6. Scaling a prim with `CollisionAPI` usually works (uniform scale on simple
   geometry), but nonuniform or time-varying scale can produce wrong bounds
   for certain collision approximations — validate with collision
   visualization. See `references/physics.md`.
7. Sensors: `play(commit=True)` before `get_data()`; velocities after play.
8. `torch` imported before the app/physics settles → CUDA context conflict
   hangs Kit. Defer torch until after boot + settle.
9. Newton registers when its extension is enabled but activates only on an
   explicit switch — verify the active engine instead of assuming PhysX.
10. `get_rigid_body_state()` does not exist in 6.0 — use
    `RigidPrim.get_world_poses()`.

## Reference routing — load one level deep when needed

Every domain of Isaac Sim work has a lead reference; detail references go one
level deeper. Load the row the moment its domain enters the task — before
writing code, not after it misbehaves. Two routing notes: a failed *run*
starts at its record and logs (the `antioch-platform` skill's running and
scenarios references) before the symptom table here; finding or publishing
assets in the shared asset catalog is `antioch-platform`'s assets reference —
the rows below are about turning files into working USD robots.

| Domain | Reference | Load when… |
|---|---|---|
| Core | `references/physics.md` | configuring PhysicsScene, Hz/solver, contact materials, joint drives, Newton vs PhysX, or debugging fall-through/instability |
| Core | `references/physics-deformables.md` | Newton `ModelBuilder.add_soft_grid` / VBD, USD-schema deformables, GPU reset behavior, or the CUDA crash boundary |
| Core | `references/usd.md` | authoring or restructuring stages, composition arcs, layered assets, or debugging property-stack overrides |
| Core | `references/troubleshooting.md` | a run hangs, crashes, black-frames, or misbehaves — symptom→fix table |
| Sensors | `references/sensors.md` | adding or debugging any sensor (camera, lidar, radar, IMU, contact), the GMO writer consumption pattern, or reading sensor data back as metrics/artifacts |
| Sensors | `references/sensors-cameras.md` | `RtxCamera`/`CameraSensor`/`TiledCameraSensor` detail, lens distortion, camera domain randomization |
| Sensors | `references/sensors-rtx.md` | lidar/radar/acoustic RTX sensors, `SUPPORTED_LIDAR_CONFIGS` vendor catalog, USDA mount attachment, custom scan patterns |
| Assets | `references/assets.md` | importing URDF/MJCF, XACRO pre-expansion, drive presets, instanceable assets — the import workflow end to end |
| Assets | `references/assets-import.md` | importer-config semantics (fix_base tri-state, per-joint overrides), transformer pipeline, robot schema — exhaustive field lists: research_search |
| Assets | `references/assets-validation.md` | before trusting an imported or hand-assembled robot — the articulation validation checklist |
| Assets | `references/assets-measurement.md` | reading `metersPerUnit`, measuring bboxes, offset-corrected placement, headless shader compatibility |
| Navigation | `references/navigation.md` | generating occupancy maps, planning or following paths, or driving wheeled/quadruped robots |
| Navigation | `references/navigation-occupancy-maps.md` | omap `Generator` and direct-projection code, height filtering, dilation buffers, ROS `map.yaml` export |
| Navigation | `references/navigation-footprints-planning.md` | runtime footprints and Z-offsets, A* planning and smoothing, oriented-footprint overlap validation |
| Navigation | `references/navigation-controllers.md` | differential/holonomic wheel control, `PolicyController` RL loading, physics-vs-baked execution on heavy stages |
| Manipulation | `references/manipulation.md` | controlling an arm to reach, grasp, transport, or place, or choosing an IK/motion stack |
| Manipulation | `references/manipulation-ik-stacks.md` | RobotPoser named poses, cuMotion RMP, PINK detail, Jacobian layout for differential IK |
| Manipulation | `references/manipulation-grasping.md` | FixedJoint and SurfaceGripper grasping, grasp frames, pick-and-place validation checkpoints |
| Manipulation | `references/manipulation-physics.md` | collider approximation, instanced colliders, misleading metrics, or deformable reset/timestep reasoning |
| Rendering | `references/rendering.md` | capturing screenshots or video, tuning lighting/tone mapping, or debugging black/flat/overexposed frames |
| Rendering | `references/rendering-capture.md` | full capture code: camera prims, AOV dtypes, per-shot capture loop, chase cameras, frame validators |
| Rendering | `references/rendering-lighting.md` | complete lighting code: baseline setups, ACES configuration, multi-layer interior recipes with fog |
| Rendering | `references/rendering-batch.md` | multi-episode/multi-shot batch loops — memory discipline, instanceable assets, throughput |
| SDG | `references/sdg.md` | generating annotated datasets or mobile-robot sensor datasets, or scaling SDG as a queued suite (`antioch suite run NAME --queue`) |
| SDG | `references/sdg-replicator.md` | writer init forms, annotator catalog detail, `rep.functional` domain randomization, throughput/DLSS knobs |
| SDG | `references/sdg-mobility-gen.md` | MobilityGen record and replay loops, output layout and state-dict format, custom robot subclassing |
