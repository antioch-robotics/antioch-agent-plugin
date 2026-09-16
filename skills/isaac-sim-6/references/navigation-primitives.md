# Navigation Primitives — Shared Substrate

Adapted from NVIDIA [`navigation-primitives/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Shared mobile-robot substrate: occupancy maps, A* planning, robot footprints, differential/holonomic kinematics, and chase-camera math.

Physical navigation keeps actuators, contacts, slip, balance, and obstacles active. Kinematic replay is for visualization and must be labeled replay. Grid occupancy is not contact.

## Limitations

- This is the shared substrate only — it does not drive robots, record SDG, or publish ROS topics; jump to the specialization references for those.
- Footprints/kinematics assume authored collider geometry and correct scene units; garbage colliders yield garbage footprints.
- Oriented-footprint PhysX checks need an initialized physics scene and a playing timeline; the grid-based check works offline but is coarser.

## Troubleshooting

| Error / symptom | Cause | Solution |
|---|---|---|
| Robot falls through floor / feet float | Spawned without measured `z_offset` | Spawn at `z = ground + compute_robot_footprint(...)["z_offset"]` |
| Path clips walls in render but not omap | Skipped oriented-footprint validation | Run `footprint_clips_grid` per waypoint (validation pipeline steps 6–7) |
| Grid full of shell/signage obstacles | Visual-bbox projection without filtering | Prefer collider-driven filtering; apply geometric filters |

Consumed by:

- [isaac-sim-robot-navigation.md](isaac-sim-robot-navigation.md): runtime navigation in custom scripts.
- [mobility-gen.md](mobility-gen.md) / [sdg.md](data-collection-sim.md): MobilityGen record/replay.
- [occupancy-map.md](occupancy-map.md): produces the `map.yaml` consumed here.

ROS 2 / Nav2 bridge setup belongs to [antioch-platform](../../antioch-platform/SKILL.md).

## Upstream helpers (source examples)

| Script | Purpose |
|---|---|
| [`scripts/occupancy_map_from_usd.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/scripts/occupancy_map_from_usd.py) | Rasterize a 2D occupancy grid from USD geometry (no PhysX required) |
| [`scripts/robot_footprint.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/scripts/robot_footprint.py) | Compute robot footprint dimensions and Z-offset from USD collision geometry |
| [`scripts/kinematics.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/scripts/kinematics.py) | Differential-drive and holonomic kinematics with pure-pursuit (no Isaac Sim dependency) |

These files are not bundled commands.

## Shared NVIDIA APIs

| Capability | Module |
|---|---|
| Occupancy maps | `isaacsim.replicator.experimental.mobility_gen.impl.occupancy_map.OccupancyMap` |
| A* path planner | `isaacsim.replicator.experimental.mobility_gen.impl.path_planner.generate_paths` |
| Runtime omap from stage | `isaacsim.asset.gen.omap.bindings._omap.Generator` |
| Robot articulation | `isaacsim.core.experimental.prims.Articulation` |
| Differential controller | `isaacsim.robot.experimental.wheeled_robots.controllers.DifferentialController` |
| Holonomic controller | `isaacsim.robot.experimental.wheeled_robots.controllers.HolonomicController` |
| Physics lifecycle | `isaacsim.core.simulation_manager.SimulationManager` |

Enable optional extension IDs through `SimulationConfig.extensions` (`isaacsim.asset.gen.omap`, `isaacsim.replicator.experimental.mobility_gen`, `isaacsim.robot.experimental.wheeled_robots`). Installed is not enabled. Classic `isaacsim.robot.wheeled_robots` is deprecated, not removed; prefer the experimental controllers for new code.

`OccupancyMap.from_ros_yaml(path)` loads a YAML+PNG pair (produced by [occupancy-map.md](occupancy-map.md)). `generate_paths(start, freespace)` takes a start cell and a freespace mask at this pin.

## Robot footprints and Z-offsets — derived at runtime

Do not hardcode footprints. Walk the articulation's collider prims and union their world-space AABBs. This stays correct when assets change.

`compute_robot_footprint(stage, robot_root)` — walk CollisionAPI prims, union their AABBs, return size, z_offset, inscribed_radius, circumscribed_radius. See [`scripts/robot_footprint.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/scripts/robot_footprint.py).

Use `inscribed_radius` when the robot can rotate freely in place (over-conservative, zero clip). Use `circumscribed_radius` only when you require zero false negatives. For non-circular robots, prefer the oriented-footprint check below over a single radius.

A world-aligned box changes with yaw; do not rotate it again. The circumscribed radius encloses the footprint; the inscribed radius omits corners.

**Spawn the robot at `z = ground + z_offset`.** Missing the Z-offset is a common cause of "robot falls through the floor" or "feet pop above ground".

### Reference values (sanity check only)

If `compute_robot_footprint` is far from these, collider authoring or scene units may be wrong. These are examples, not spawn constants:

| Robot | Example size (m) | Example z_offset | Example inscribed r |
|---|---|---|---|
| Spot | ~1.08 × 0.44 × 0.55 | ~0.69 | ~0.22 |
| Spot + arm | ~1.10 × 0.40 × 1.20 | ~0.69 | ~0.20 |
| Nova Carter | track_w=0.499, wheel_r=0.14 | ~0.0 | ~0.25 |
| VSVXL | ~2.52 × 1.72, 6-wheel diff | ~0.0 | ~0.86 |
| Jetbot | wheel_base=0.1125, wheel_r=0.03 | ~0.02 | ~0.06 |
| Kaya (holonomic) | wheel_base=0.10, wheel_r=0.04 | ~0.02 | ~0.10 |
| H1 (humanoid) | — | ~1.05 | ~0.20 |

## Occupancy map from USD (direct projection)

Use when you need a runtime omap and don't already have a `map.yaml`. For the canonical `map.yaml` workflow consumed by MobilityGen, use [occupancy-map.md](occupancy-map.md).

`occupancy_map_from_usd(stage, x_range, y_range, resolution, z_cutoff, colliders_only)` — rasterize USD geometry into a 2D uint8 grid. See [`scripts/occupancy_map_from_usd.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/scripts/occupancy_map_from_usd.py).

Declare map resolution, origin, bounds, vertical obstacle band, and unknown-space policy before generating a grid.

### Obstacle filtering

Two strategies, in order of preference:

**A. Collider-driven (preferred when assets have authored colliders).** Iterate only prims with `UsdPhysics.CollisionAPI`. This already excludes visual-only geometry (signage, decals, light cones, debug arrows) without a filter list.

```python
from pxr import UsdPhysics

for prim in Usd.PrimRange(stage.GetPrimAtPath("/World")):
    if not prim.HasAPI(UsdPhysics.CollisionAPI):
        continue
    enabled = prim.GetAttribute("physics:collisionEnabled")
    if enabled and enabled.Get() is False:
        continue
    # rasterize this prim's AABB into the grid
```

**B. Visual-bbox + filter list (fallback for scenes without colliders).** Naive bbox projection fills the grid with shell, zones, and signage. Filter names for the current scene; the lists below are warehouse-scene examples, not a universal skip catalog:

```python
SKIP_SCOPES = {"GroundPlane", "Looks", "Lighting", "Render", "PushGraph", "DomeLight", "DemoCamera"}
SKIP_PREFIXES = ("Floor", "FL_", "FR_", "Exit_", "Hum")
SKIP_CHILDREN = ("Zones", "signage")
```

Geometric filters (apply after either strategy; thresholds are starting points):
- Skip area far larger than the mapped facility (shell / zone assemblies)
- Skip height below floor-marking thickness (example: < 0.1 m)
- Skip `Z_min` above the robot height band
- Skip `Z_max` below `fp["z_offset"]` plus a small clearance (drive-over)

### Buffer sizing — derived from footprint

The buffer is `circumscribed_radius + safety_margin`, not a magic constant. Convert nonnegative clearance to cells with `ceil(clearance_m / resolution_m)`. Example safety margins (tuning starting points):

| Context | Example safety margin |
|---|---|
| Open corridor, smooth control | 0.10 m |
| Aisle navigation, cluttered | 0.30 m |
| Cluttered + non-zero yaw error | 0.50 m |

```python
fp = compute_robot_footprint(stage, "/World/Robot")
buffer_m = fp["circumscribed_radius"] + 0.30  # aisle example
buffer_cells = int(np.ceil(buffer_m / RESOLUTION))
```

For a Spot-sized footprint (`circumscribed_radius ≈ 0.58 m`) plus 0.30 m this yields ~0.88 m, not a 1.5 m blanket value. With the oriented-footprint check you recover navigable space the circular proxy threw away.

## A* path planning

Erode by `inscribed_radius` (fast, conservative). Then validate the smoothed path with an oriented-footprint collision check, which recovers the navigable space the inscribed-radius erosion threw away.

```python
from scipy.ndimage import binary_erosion
import numpy as np

fp = compute_robot_footprint(stage, "/World/Robot")
kernel_r = int(np.ceil(fp["inscribed_radius"] / RESOLUTION))
kernel = np.zeros((2 * kernel_r + 1, 2 * kernel_r + 1), dtype=bool)
for dy in range(-kernel_r, kernel_r + 1):
    for dx in range(-kernel_r, kernel_r + 1):
        if dx * dx + dy * dy <= kernel_r * kernel_r:
            kernel[dy + kernel_r, dx + kernel_r] = True

navigable = grid == 0
eroded = binary_erosion(navigable, structure=kernel)
# Standard A* over `eroded` (heapq-based), or generate_paths on a freespace mask
```

### Oriented-footprint collision check (recommended for non-circular robots)

**1. Against PhysX scene (after physics is initialized and playing):** use `get_physx_scene_query_interface().overlap_box`. Returns hit count; >0 means clip. Exclude the queried robot, classify the support surface, and verify filter/callback semantics. PhysX `overlap_box` takes XYZW (`imaginary, real`); Isaac Sim tensor APIs commonly use WXYZ.

```python
import carb
from omni.physx import get_physx_scene_query_interface
from pxr import Gf


def footprint_clips(x: float, y: float, yaw: float, fp: dict, z_query: float = 0.2) -> bool:
    half = carb.Float3(fp["size"][0] / 2, fp["size"][1] / 2, fp["size"][2] / 2)
    origin = carb.Float3(x, y, z_query + fp["size"][2] / 2)
    rot = Gf.Rotation(Gf.Vec3d(0, 0, 1), np.degrees(yaw)).GetQuat()
    quat = carb.Float4(*rot.GetImaginary(), rot.GetReal())  # XYZW for this PhysX query
    hits = get_physx_scene_query_interface().overlap_box(half, origin, quat, lambda h: True, anyHit=True)
    return hits > 0
```

**2. Against the rasterized omap (no PhysX needed):** stamp the rotated rectangle onto the obstacle grid and AND with the occupied mask.

```python
import cv2


def footprint_clips_grid(px: int, py: int, yaw: float, fp: dict, grid: np.ndarray) -> bool:
    w_cells = fp["size"][0] / RESOLUTION
    h_cells = fp["size"][1] / RESOLUTION
    rect = ((px, py), (w_cells, h_cells), np.degrees(yaw))
    pts = cv2.boxPoints(rect).astype(np.int32)
    mask = np.zeros_like(grid, dtype=np.uint8)
    cv2.fillPoly(mask, [pts], 1)
    return bool(np.any((grid > 0) & (mask > 0)))
```

### Validation pipeline

1. `compute_robot_footprint(stage, robot_root)` — get size, z_offset, radii.
2. Rasterize obstacles onto grid (0.25 m resolution is a typical starting point).
3. Binary erode with circular kernel of radius = `ceil(inscribed_radius / resolution)`.
4. A* pathfind on eroded grid.
5. Catmull-Rom smooth the raw path; assign yaw = `atan2(dy, dx)` along the curve.
6. For every smoothed waypoint, run `footprint_clips_grid(...)` (or `footprint_clips(...)` against PhysX). Reject the path on any hit.
7. If a single waypoint fails, snap to the nearest navigable cell and re-validate. If multiple fail, re-plan with a larger erosion kernel (`circumscribed_radius`).

Skipping steps 6–7 produces paths that look fine on the omap but clip walls in simulation — especially on rectangular robots cornering through aisles. Grid checks remain approximate; use the physical controller for a physical claim.

## Kinematics helpers

[`scripts/kinematics.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/scripts/kinematics.py) — standalone differential-drive and holonomic kinematics with pure-pursuit path following. No Isaac Sim dependency; works offline for planning or inside a simulation loop.

Key exports:
- `differential_forward(v, ω, params)` → `(vL, vR)` wheel rad/s
- `differential_inverse(vL, vR, params)` → `(v, ω)` body twist
- `holonomic_forward(vx, vy, ω, params)` → 3 wheel velocities (120°-spaced)
- `holonomic_inverse(wheel_vels, params)` → `(vx, vy, ω)`
- `pure_pursuit_step(pos, yaw, path, config)` — PD-steered differential path follower
- `holonomic_path_step(pos, yaw, path, config)` — strafing holonomic path follower

## Differential drive kinematics

For `motion_generation` controller composition, see [motion-generation.md](motion-generation.md); this section owns the geometry, wheel kinematics, and navigation tuning.

```python
# Wheel velocities from body twist
vL = (vx - omega * track_w / 2) / wheel_r
vR = (vx + omega * track_w / 2) / wheel_r
```

PD steering (example starting points used on Nova Carter / VSVXL-class platforms; retune from measured heading error):
- KP=2.5, KD=1.2, MAX_W=1.5 rad/s
- Waypoint tolerance: 4.0 m
- Speed reduction near waypoints and during large heading errors
- Out-of-bounds watchdog example: |Z| > 50 or |X|/|Y| > 500 → mark dead

VSVXL-class example (tuning starting points, not a required set):
- ω = v / r (0.15 m wheels → 10 rad/s for 1.5 m/s)
- Physics: dt=1/120 s, substeps=4 (effective 480 Hz) as a high-rate starting point
- PID heading: Kp=2.0, Ki=0.1, Kd=0.5, max angular 1.0 rad/s
- Aisle speed: 0.8 m/s; corridor speed: 1.5 m/s
- Corridor buffer: 0.5 m; aisle buffer: 0.2 m + reduced speed

## Holonomic / mecanum (Kaya, AMR)

Use experimental `HolonomicController` with wheel positions, orientations (WXYZ), and mecanum angles from the robot USD. Apply via DOF velocity targets on `Articulation`.

2D MobilityGen action `[lin, ang]` → 3D holonomic command `[forward, lateral=0, yaw]`. See [mobility-gen.md](mobility-gen.md) for robot subclass patterns.

## DifferentialController + Articulation

```python
from isaacsim.robot.experimental.wheeled_robots.controllers import DifferentialController
from isaacsim.core.experimental.prims import Articulation
import numpy as np

robot = Articulation("/World/Robot")
robot.initialize_cpp_data_view()

dc = DifferentialController(wheel_radius=0.15, wheel_base=1.52)  # example geometry
wheel_vels = dc.forward(np.array([linear_speed, angular_speed]))

vel_targets = np.zeros((1, num_dofs))
for i in left_wheel_indices:
    vel_targets[0, i] = wheel_vels[0]
for i in right_wheel_indices:
    vel_targets[0, i] = wheel_vels[1]
robot.set_dof_velocity_targets(vel_targets)
```

`set_dof_velocity_targets` is batched `(N, num_dofs)`. Wheel indices, input units, and control period come from the actual robot.

## Ackermann / bicycle kinematics

For RC-car/forklift-style steering, plan with a curvature limit and smooth both speed and steering-rate. Derive steering from wheel base and curvature, then derive yaw rate from speed and the resulting steering angle; keep the controller-specific implementation with the robot integration rather than embedding a partial snippet here.

Use a larger lookahead or Dubins-style path when the course has tight turns. For `motion_generation` controller composition, see [motion-generation.md](motion-generation.md).

## World transform extraction

`BBoxCache.ComputeWorldBound()` returns local bounds for some Cube prims — wrong for world position if you treat the range as a world translation. For authored world position:

```python
xf_cache = UsdGeom.XformCache(Usd.TimeCode.Default())
world_mat = xf_cache.GetLocalToWorldTransform(prim)
position = world_mat.ExtractTranslation()
```

For **simulated** position (Articulation runtime, not authored):

```python
pos_wp, quat_wp = robot.get_world_poses()
pos = pos_wp.numpy()[0]  # shape (3,)
quat = quat_wp.numpy()[0]  # shape (4,) [w,x,y,z]
```

`XformCache` reads authored USD and can lag physics writeback. `Articulation.get_world_poses()` reads simulated state (batched WXYZ). Clear/rebuild caches after edits.

## Look-at chase camera

Use look-at rather than ad-hoc rotation matrices for chase cameras. This aims a viewport or authored USD camera; it does not replace sensor cameras or Replicator products ([sensors.md](isaac-sim-sensor.md), [isaac-camera.md](isaac-camera.md)).

USD optical cameras look along −Z. Classic Isaac camera `camera_axes="world"` is +X forward / +Z up; ROS camera axes are +Z forward / −Y up. Do not mix those conventions.

```python
from pxr import Gf


def look_at(eye, target, up=Gf.Vec3d(0, 0, 1)):
    fwd = (target - eye).GetNormalized()
    right = Gf.Vec3d.GetCross(fwd, up).GetNormalized()
    cam_up = Gf.Vec3d.GetCross(right, fwd)
    # USD camera: -Z is forward
    return Gf.Matrix4d(right[0], right[1], right[2], 0, cam_up[0], cam_up[1], cam_up[2], 0, -fwd[0], -fwd[1], -fwd[2], 0, eye[0], eye[1], eye[2], 1)
```

Example camera offsets (starting points):
- **Chase**: 4 m behind robot, 2.5 m up, looking at robot center
- **Overhead**: 10 m up, looking straight down
- **POV**: at robot front (example 1.26 m forward), 0.8 m height

### Degenerate up-vector

When camera looks straight down (fwd ≈ 0,0,−1), `cross(fwd, up=(0,0,1))` is zero → broken matrix → blank render. Always fallback:

```python
up = Gf.Vec3d(0, 0, 1)
if abs(fwd * up) > 0.99:
    up = Gf.Vec3d(0, 1, 0)
```

## Common gotchas

1. **Feet/wheels below origin**: many robots (Spot ~0.69 m, H1 ~1.05 m as examples) have their articulation origin above the ground contact. Always call `compute_robot_footprint(stage, root)` and spawn at `z = ground + fp["z_offset"]`.
2. **Instancing**: instance proxies are read-only. Flatten or de-instance only the subtree that needs unique state; do not treat instanceability as a headless-architecture mandate.
3. **OmniGraph time jumps**: jumping past `endTimeCode` can crash PushGraph camera animation nodes.
4. **`next_update_async`**: use `omni.kit.app.get_app().next_update_async()`.
5. **Per-frame yaw smoothing**: `current_yaw += yaw_diff * 0.15` is an example blend, not a required gain.
6. **Frame numbering for ffmpeg**: sequential `frame_0000.png`, `frame_0001.png`... ffmpeg skips gaps. This is encode hygiene, not physics evidence.
7. **Unlit routes**: A* paths through dark corridors can capture as black frames. Light the route or use a sensor/Replicator product with explicit lights. Example: SphereLights at Z=3 m, intensity 800, radius 0.3 — retune per renderer.

## Specialization references

| Goal | Reference |
|---|---|
| Drive a robot through a scene in real time (RL policy, physics, baked replay) | [isaac-sim-robot-navigation.md](isaac-sim-robot-navigation.md) |
| Record a trajectory and re-render it with sensors for SDG | [mobility-gen.md](mobility-gen.md) |
| Generate the `map.yaml` from USD | [occupancy-map.md](occupancy-map.md) |
| Publish/subscribe nav topics to ROS 2 / Nav2 | [antioch-platform](../../antioch-platform/SKILL.md) |
