# Navigation Primitives — Shared Substrate

Adapted from NVIDIA [`navigation-primitives/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/SKILL.md) (Apache-2.0).

[Isaac Sim task index](../SKILL.md#domain-references) · [Antioch startup](../../antioch-platform/references/simulation-code.md)

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
- [mobility-gen.md](mobility-gen.md) / [data collection](data-collection-sim.md): MobilityGen record/replay.
- [occupancy-map.md](occupancy-map.md): produces the `map.yaml` consumed here.

ROS 2 / Nav2 bridge setup belongs to [ros2](../../ros2/SKILL.md).

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
| Breadth-first reachability search | `isaacsim.replicator.experimental.mobility_gen.impl.path_planner.generate_paths` |
| Runtime omap from stage | `isaacsim.asset.gen.omap.bindings._omap.Generator` |
| Robot articulation | `isaacsim.core.experimental.prims.Articulation` |
| Differential controller | `isaacsim.robot.experimental.wheeled_robots.controllers.DifferentialController` |
| Holonomic controller | `isaacsim.robot.experimental.wheeled_robots.controllers.HolonomicController` |
| Physics lifecycle | `isaacsim.core.simulation_manager.SimulationManager` |

Enable optional extension IDs through `SimulationConfig.extensions` (`isaacsim.asset.gen.omap`, `isaacsim.replicator.experimental.mobility_gen`, `isaacsim.robot.experimental.wheeled_robots`). Installed is not enabled. Classic `isaacsim.robot.wheeled_robots` is deprecated, not removed; prefer the experimental controllers for new code.

`OccupancyMap.from_ros_yaml(path)` loads a YAML+PNG pair (produced by [occupancy-map.md](occupancy-map.md)). `generate_paths(start, freespace)` takes a start cell and a freespace mask, then runs breadth-first search over reachable cells. It does not implement A*. MobilityGen can sample a reachable destination and walk the resulting search tree.

## Robot footprints and Z-offsets — derived at runtime

Do not hardcode footprints. Walk the articulation's collider prims and union their world-space AABBs. This stays correct when assets change. Traverse with `Usd.PrimRange(root, Usd.TraverseInstanceProxies())`, since a plain range skips instanced colliders such as Nova Carter's wheels, and check each box before the union: two Nova Carter caster cylinders reported 100 m bounds.

`compute_robot_footprint(stage, robot_root)` — walk CollisionAPI prims, union their AABBs, return size, z_offset, inscribed_radius, circumscribed_radius. See [`scripts/robot_footprint.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/scripts/robot_footprint.py).

The circumscribed radius encloses the footprint and is conservative for
rotation. The inscribed radius omits corners and cannot guarantee clearance.
For non-circular robots, validate the oriented footprint along the path.

For the oriented checks below, measure collider bounds in the robot root frame, or with a level robot at yaw zero, and keep that box fixed. A world-aligned box measured at the current yaw must not be rotated again. Store its center offset from the robot origin as well as its size; transform that offset with the planned robot pose before querying.

**Spawn the robot at `z = ground + z_offset`.** Missing the Z-offset is a common cause of "robot falls through the floor" or "feet pop above ground".

### Geometry examples

The upstream helper uses Nova Carter wheel radius 0.14 m and track width 0.499 m, and Jetbot wheel radius 0.03 m and track width 0.1125 m as example configurations. Check the selected robot asset and joint frames before using either. An added arm, sensor rack, or payload can change footprint and clearance without changing wheel geometry.

Upstream also records these asset-specific sanity values. They help spot a unit or collider mistake; a different asset revision, pose, or payload can legitimately change them:

| Robot | Example dimensions or wheel geometry | Z offset | Inscribed radius |
|---|---|---|---|
| Spot | 1.08 × 0.44 × 0.55 m | 0.69 m | 0.22 m |
| Spot with arm | 1.10 × 0.40 × 1.20 m | 0.69 m | 0.20 m |
| VSVXL | 2.52 × 1.72 m, six-wheel differential drive | About 0 m | 0.86 m |
| Kaya | 0.10 m wheel base, 0.04 m wheel radius | 0.02 m | 0.10 m |
| H1 | Measure the selected humanoid pose | 1.05 m | 0.20 m |

## Occupancy map from USD (direct projection)

Use when you need a runtime omap and don't already have a `map.yaml`. For the canonical `map.yaml` workflow consumed by MobilityGen, use [occupancy-map.md](occupancy-map.md).

`occupancy_map_from_usd(stage, x_range, y_range, resolution, z_cutoff, colliders_only)` — rasterize USD geometry into a 2D uint8 grid. See [`scripts/occupancy_map_from_usd.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/navigation-primitives/scripts/occupancy_map_from_usd.py).

This helper returns a uint8 grid with **0 = free and 255 = occupied**. The occupancy-map generator has a separate 0/1/2 state convention; convert it explicitly before using these snippets. Declare map resolution, origin, row direction, bounds, vertical obstacle band, and unknown-space policy before generating a grid.

### Obstacle filtering

Two strategies, in order of preference:

**A. Collider-driven (preferred when assets have authored colliders).** Iterate only prims with `UsdPhysics.CollisionAPI`. This already excludes visual-only geometry (signage, decals, light cones, debug arrows) without a filter list.

```python
from pxr import Usd, UsdPhysics

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
import math

fp = compute_robot_footprint(stage, "/World/Robot")
buffer_m = fp["circumscribed_radius"] + 0.30  # aisle example
buffer_cells = math.ceil(buffer_m / RESOLUTION)
```

For a Spot-sized footprint (`circumscribed_radius ≈ 0.58 m`) plus 0.30 m this yields ~0.88 m, not a 1.5 m blanket value. With the oriented-footprint check you recover navigable space the circular proxy threw away.

## A* path planning

Inscribed-radius erosion can generate candidate paths but leaves corner
collisions possible. Validate the smoothed path's swept oriented footprint;
circumscribed-radius erosion is a more conservative candidate filter.

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
# Use A* for a goal-directed path, or generate_paths for a reachable-cell search tree.
```

### Oriented-footprint collision check (recommended for non-circular robots)

**1. Against PhysX scene (after physics is initialized and playing):** use `get_physx_scene_query_interface().overlap_box`. Classify hits to exclude the queried robot and its intended support surface. The helper below accepts that task-specific predicate and queries the fixed local box at its planned world center. PhysX `overlap_box` takes XYZW (`imaginary, real`); Isaac Sim tensor APIs commonly use WXYZ.

```python
import math
import carb
from omni.physx import get_physx_scene_query_interface
from pxr import Gf


def footprint_clips(center_world, yaw, size_local, is_obstacle):
    half = carb.Float3(*(value / 2 for value in size_local))
    rot = Gf.Rotation(Gf.Vec3d(0, 0, 1), math.degrees(yaw)).GetQuat()
    quat = carb.Float4(*rot.GetImaginary(), rot.GetReal())  # XYZW
    blocked = False

    def on_hit(hit):
        nonlocal blocked
        if is_obstacle(hit):
            blocked = True
            return False  # Stop after the first relevant obstacle.
        return True

    get_physx_scene_query_interface().overlap_box(half, carb.Float3(*center_world), quat, on_hit, anyHit=False)
    return blocked
```

**2. Against the rasterized omap (no PhysX needed):** stamp the rotated rectangle onto the obstacle grid. This example uses ROS image orientation: row 0 is maximum world Y, columns increase with X, and yaw is counterclockwise in world XY. Pass the box center as `(col, row)`, not the robot origin. For a grid whose rows increase with world Y, use positive yaw instead.

```python
import cv2
import numpy as np


def footprint_clips_grid(col, row, yaw, size_local, grid, resolution):
    width = size_local[0] / resolution
    height = size_local[1] / resolution
    rect = ((col, row), (width, height), -np.degrees(yaw))
    points = cv2.boxPoints(rect)
    # Treat unknown space outside the map as blocked.
    if points[:, 0].min() < 0 or points[:, 1].min() < 0 or points[:, 0].max() >= grid.shape[1] or points[:, 1].max() >= grid.shape[0]:
        return True
    mask = np.zeros_like(grid, dtype=np.uint8)
    cv2.fillPoly(mask, [np.rint(points).astype(np.int32)], 1)
    return bool(np.any((grid > 0) & (mask > 0)))
```

Grid rasterization is approximate. Include a resolution-dependent safety margin, and sample between waypoints densely enough to check the swept footprint.

### Validation pipeline

1. Measure fixed robot-frame collider bounds, center offset, Z offset, and radii. Use the upstream world-bound helper at yaw zero only; do not feed a yawed world AABB into an oriented check.
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

Tune steering gains, speed, lookahead, and waypoint tolerances from measured
tracking error and the actual robot. Check timestep sensitivity before
increasing controller gains.

For an initial differential-drive controller experiment, upstream uses heading gains `Kp=2.5`, `Kd=1.2` and an angular-speed cap of 1.5 rad/s. These values assume its error units and update cadence. Reduce speed near sharp turns, choose a waypoint tolerance smaller than the goal region, and measure tracking error, slip, and stopping distance. Do not copy a multi-meter demo tolerance into a precise navigation test.

The upstream long-route demo used a 4 m waypoint tolerance and stopped a robot as out of bounds when `abs(z) > 50` or `abs(x) > 500` or `abs(y) > 500`. Keep those values only for a scene and goal scale that justify them. A precise arrival check needs a much smaller tolerance.

Its VSVXL recipe used 0.15 m wheels, heading PID gains `(2.0, 0.1, 0.5)`, a 1 rad/s turn cap, speeds of 0.8 m/s in aisles and 1.5 m/s in corridors, and clearance margins of 0.2 m and 0.5 m respectively. It described 1/120 s stepping with four substeps. Verify how the chosen stepping owner applies those settings; measure simulated time and tracking error before calling that 480 Hz. These are upstream tuning examples, not measurements reproduced on Antioch.

## Holonomic / mecanum (Kaya, AMR)

Use experimental `HolonomicController` with wheel positions, orientations (WXYZ), and mecanum angles from the robot USD. Apply via DOF velocity targets on `Articulation`.

2D MobilityGen action `[lin, ang]` → 3D holonomic command `[forward, lateral=0, yaw]`. See [mobility-gen.md](mobility-gen.md) for robot subclass patterns.

## DifferentialController + Articulation

```python
from isaacsim.robot.experimental.wheeled_robots.controllers import DifferentialController
from isaacsim.core.experimental.prims import Articulation
import numpy as np

from isaacsim.core.experimental.utils import app as app_utils

app_utils.play(commit=True)  # Tensor views need initialized, playing physics.
robot = Articulation("/World/Robot")

dc = DifferentialController(wheel_radius=0.15, wheel_base=1.52)  # example geometry
wheel_vels = dc.forward(np.array([linear_speed, angular_speed]))

vel_targets = np.zeros((1, num_dofs))
for i in left_wheel_indices:
    vel_targets[0, i] = wheel_vels[0]
for i in right_wheel_indices:
    vel_targets[0, i] = wheel_vels[1]
robot.set_dof_velocity_targets(vel_targets)
```

`initialize_cpp_data_view()` is optional and only needed by C++ consumers of that read-only view. `set_dof_velocity_targets` is batched `(N, num_dofs)`. Wheel indices, input units, and control period come from the actual robot.

## Ackermann / bicycle kinematics

For RC-car/forklift-style steering, plan with a curvature limit and smooth both speed and steering-rate. Derive steering from wheel base and curvature, then derive yaw rate from speed and the resulting steering angle; keep the controller-specific implementation with the robot integration rather than embedding a partial snippet here.

Use a larger lookahead or Dubins-style path when the course has tight turns. For `motion_generation` controller composition, see [motion-generation.md](motion-generation.md).

## World transform extraction

`BBoxCache.ComputeWorldBound()` returns a transformed bounding box. Call `ComputeAlignedRange()` to get world-axis-aligned bounds; its untransformed range is not the world range. Neither the bound center nor its minimum is necessarily the prim origin. For the authored world position of the origin:

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

Use look-at rather than ad-hoc rotation matrices for chase cameras. This aims a viewport or authored USD camera; it does not replace sensor cameras or Replicator products ([sensors](isaac-sim-sensor.md), [isaac-camera.md](isaac-camera.md)).

USD optical cameras look along −Z. Classic Isaac camera `camera_axes="world"` is +X forward / +Z up; ROS camera axes are +Z forward / −Y up. Do not mix those conventions.

```python
from pxr import Gf


def look_at(eye, target, up=Gf.Vec3d(0, 0, 1)):
    fwd = (target - eye).GetNormalized()
    if abs(fwd * up) > 0.99:
        up = Gf.Vec3d(0, 1, 0)
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
4. **Awaiting a Kit frame**: get the Kit app before calling `next_update_async()`: `await omni.kit.app.get_app().next_update_async()`. Do not call it on the `omni.kit.app` module.
5. **Per-frame yaw smoothing**: `current_yaw += yaw_diff * 0.15` is an example blend, not a required gain.
6. **Frame numbering for ffmpeg**: use sequential `frame_00000.png`, `frame_00001.png`, … with `%05d`. The image-sequence reader can stop at a gap. Use timestamps to choose playback rate; encoding alone does not prove correct simulation timing.
7. **Unlit routes**: A* paths through dark corridors can capture as black frames. Light the route or use a sensor/Replicator product with explicit lights. Example: SphereLights at Z=3 m, intensity 800, radius 0.3 — retune per renderer.

## Specialization references

| Goal | Reference |
|---|---|
| Drive a robot through a scene in real time (RL policy, physics, baked replay) | [isaac-sim-robot-navigation.md](isaac-sim-robot-navigation.md) |
| Record a trajectory and re-render it with sensors for SDG | [mobility-gen.md](mobility-gen.md) |
| Generate the `map.yaml` from USD | [occupancy-map.md](occupancy-map.md) |
| Publish/subscribe nav topics to ROS 2 / Nav2 | [ros2](../../ros2/SKILL.md) |
