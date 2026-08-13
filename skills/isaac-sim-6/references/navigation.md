# Navigation — Isaac Sim 6.0.1 on Antioch

Detail reference for the `isaac-sim-6` skill.

Navigation code is authored locally (no simulator installed) and executed
remotely on Antioch GPU machines running Isaac Sim 6.0.1 / Kit 110.1.2.
Simulation commands can stream the remote GUI, but there is no local Kit or
local stage. Verify navigation through recorded metrics, telemetry, and
artifacts instead of relying on the viewport alone.

Shared substrate lives elsewhere — point, don't duplicate: `SKILL.md` owns
the lazy-import invariant, the 6.0 namespace map, boot/step/readback
ordering, and the XformCache trap; `antioch-platform` owns dispatch. One
scenario process per execution slot: there is no local Kit to kill and
relaunch — fix the code and re-dispatch.

## Module map (Isaac Sim 6.0.1)

Never `omni.isaac.*`. Exact current paths:

| Capability | Module |
|---|---|
| Occupancy map from USD (extension-gated PhysX raycasts) | `isaacsim.asset.gen.omap.bindings._omap.Generator` after enabling `isaacsim.asset.gen.omap` |
| Load ROS map.yaml + PNG | `isaacsim.replicator.experimental.mobility_gen.OccupancyMap` (package root) |
| BFS path planner | `isaacsim.replicator.experimental.mobility_gen.impl.path_planner.generate_paths` |
| Robot articulation | `isaacsim.core.experimental.prims.Articulation` |
| Differential controller | `isaacsim.robot.experimental.wheeled_robots.controllers.DifferentialController` |
| Holonomic controller | `isaacsim.robot.wheeled_robots.controllers.holonomic_controller.HolonomicController` |
| RL policy execution | `isaacsim.robot.policy.examples.controllers.PolicyController` |
| Physics lifecycle | `isaacsim.core.simulation_manager.SimulationManager` |

## The navigation workflow

1. **Get an occupancy map** — generate from the USD stage (two programmatic
   paths below) or load a previously exported `map.yaml` pair.
2. **Derive the robot footprint at runtime** — walk collider prims, never
   hardcode dimensions. This also yields the Z-offset the spawn needs.
3. **Plan and validate the path** — erode, plan, smooth, then validate every
   smoothed waypoint with an oriented-footprint overlap check.
4. **Choose the execution model** — physics, baked transforms, or per-frame
   transforms, driven by stage weight and whether dynamics matter.
5. **Drive the robot** — `DifferentialController`/holonomic on the
   experimental `Articulation`, or an RL policy via `PolicyController`.

## Occupancy maps — two programmatic paths

The GUI occupancy-map workflow does not exist here. Both paths are code.

| | Direct USD projection (first-class fallback) | Extension-gated `Generator` |
|---|---|---|
| Mechanism | Rasterize authored world AABBs onto a grid | PhysX raycasts over collision geometry |
| Requires | Nothing — works pre-play, deterministic | Enable `isaacsim.asset.gen.omap`; colliders authored on obstacles; **timeline playing** |
| Best for | Collider-less scenes, prototypes, placeholder geometry; live-proven fallback | When the optional extension loads and collision approximations are the source of truth |
| Failure mode | Noise (shells, signage, floor markings) without filters | ModuleNotFoundError before enablement, or an all-free map with stopped timeline/missing colliders |

Both paths produce the same grid; export it as ROS `map.yaml` + PNG, which
`OccupancyMap.from_ros_yaml()` and the A* planner consume. Cell size, height
band, and buffer sizing are decisions, not constants — full code and the
sizing tables: `references/navigation-occupancy-maps.md`.

## Robot footprints — derive at runtime rather than hardcode

Walk the articulation's collider prims and union their world-space AABBs —
hardcoded dimensions silently break when the asset changes.
The result carries four things the rest of navigation consumes: footprint
size, **Z-offset** (how far the origin sits above the lowest collider),
inscribed radius (safe at any yaw), and circumscribed radius (worst-case
yaw). Full implementation: `references/navigation-footprints-planning.md`.

- **Spawn at `z = ground + z_offset`.** A missing Z-offset is the #1
  cause of "robot falls through the floor" / "feet pop above ground" — many
  robots have their articulation origin well above ground contact (quadruped
  ~0.7 m, humanoid ~1.0 m).
- Erode the grid by the **inscribed** radius for planning; size the dilation
  buffer as `circumscribed_radius + safety_margin` (margin table:
  `references/navigation-footprints-planning.md`).

## Path planning and validation

Keep the whole pipeline — skipping validation produces paths that look
fine on the map but clip walls at runtime, especially rectangular robots
cornering through aisles:

1. Compute the footprint (size, z_offset, radii).
2. Rasterize obstacles (0.10–0.25 m/cell typical).
3. Binary-erode the free space with a circular kernel of
   `inscribed_radius / resolution`.
4. Plan over the eroded grid — `generate_paths` runs BFS from a start cell
   (no goal) and returns a tree you `unroll_path(end)` on, good for
   any-reachable-goal and SDG sampling; for goal-directed routing use a
   plain heapq A* instead.
5. Smooth (Catmull-Rom); assign each waypoint `yaw = atan2(dy, dx)`.
6. **Validate every smoothed waypoint** with an oriented-footprint check —
   PhysX `overlap_box` after sim init, or rotated-rectangle stamping against
   the raster grid with no PhysX needed. Reject the path on any hit.
7. One bad waypoint → snap to nearest navigable cell and re-validate.
   Many bad → re-plan with a `circumscribed_radius` erosion kernel.

Both overlap-check implementations: `references/navigation-footprints-planning.md`.

## Driving the robot

| Robot | Controller |
|---|---|
| Differential drive (two wheel groups) | `DifferentialController(wheel_radius, wheel_base)` → per-wheel velocity targets on the experimental `Articulation` |
| Holonomic / mecanum | `HolonomicController` with wheel positions/orientations extracted from the robot USD |
| RL-trained (policy checkpoint) | `PolicyController` — load the policy at runtime, step it each physics tick |

Set velocity targets **after** `play(commit=True)` — PhysX ignores
pre-play commands (boot ordering: `isaac-sim-6`). Controller wiring,
policy loading, and steering parameters: `references/navigation-controllers.md`.

## Execution model on heavy stages

| Model | How | Use when |
|---|---|---|
| Physics | Timeline playing, controller drives joints each step | Dynamics matter (contact, slip, policy rollouts) and the stage is light |
| Baked timeSamples | Pre-compute the trajectory, write xform timeSamples, scrub timeline | Static playback of a known route on a light stage |
| Per-frame transform | `set_world_poses()` along the trajectory, one step per frame | **Heavy stages (50K+ prims)** — the OOM-safe default |

Two failure modes decide for you:

- Physics on a stage with tens of thousands of rigid/collision bodies
  **hangs**. Strip `RigidBodyAPI`/`CollisionAPI` from every non-robot prim.
- Baked timeSamples + RT rendering on a 50K+ prim stage **OOMs the GPU**
  (`VkResult: ERROR_OUT_OF_DEVICE_MEMORY`). Switch to per-frame transforms.

Both patterns: `references/navigation-controllers.md`.

## Gotchas — symptom → cause → fix

- **Robot falls through floor or floats** → missing Z-offset →
  `compute_robot_footprint`, spawn at `ground + fp["z_offset"]`.
- **Map comes back all-free from `Generator`** → timeline not playing
  (raycasts need it) or obstacles lack `CollisionAPI` → `play(commit=True)`
  before `generate2d()`; else use direct projection.
- **Map full of phantom obstacles** → direct projection over visual bboxes
  without filters → iterate collider prims only, or apply skip lists +
  geometric filters (see occupancy-maps reference).
- **Path fine on map, robot clips a rack** → skipped pipeline steps 6–7 →
  oriented-footprint-validate every smoothed waypoint.
- **Sim hangs on a big warehouse stage** → physics authored on thousands of
  static prims → strip non-robot rigid/collision APIs.
- **`VkResult: ERROR_OUT_OF_DEVICE_MEMORY` mid-run** → baked transforms +
  heavy stage → per-frame transform model.
- **Pose reads never change while the robot moves** → reading authored USD
  via `XformCache` instead of simulated state →
  `Articulation.get_world_poses()` (the XformCache trap, `isaac-sim-6`).
- **Velocity commands do nothing** → set before `play()` → set after
  `play(commit=True)` (boot ordering, `isaac-sim-6`).
- **A run hung or crashed** → nothing to recover locally; every Antioch
  execution is a fresh process in its own slot → fix and re-dispatch with
  `antioch scenario run` (`antioch-platform`).

## References (load one level deep when needed)

- `references/navigation-occupancy-maps.md` — `Generator` and direct-projection code,
  collider-vs-visual filtering, height bands, resolution/buffer sizing
  tables, ROS `map.yaml` + PNG export, coordinate conventions. Load when
  generating or exporting an occupancy map.
- `references/navigation-footprints-planning.md` — `compute_robot_footprint`,
  reference-value sanity table, erosion/buffer math, A* + smoothing, and
  both oriented-footprint overlap checks (PhysX and grid). Load when
  planning or validating paths.
- `references/navigation-controllers.md` — `DifferentialController` and
  holonomic wiring on the experimental `Articulation`, steering parameters,
  `PolicyController` loading, physics/baked/per-frame execution patterns,
  GPU OOM avoidance. Load when driving the robot or choosing an execution
  model.
