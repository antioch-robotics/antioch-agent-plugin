# Robot Footprints, A* Planning, and Path Validation

Everything downstream of the occupancy map depends on the robot footprint.
Derive it at runtime — hardcoded dimensions silently break when the asset
changes. All simulator imports stay inside function bodies (Rule #1,
`isaac-sim-6`).

## `compute_robot_footprint`

Walk the prims tagged with `UsdPhysics.CollisionAPI` under the robot root
and union their world-space AABBs. Falls back to the root prim's bound if
no colliders are authored.

```python
def compute_robot_footprint(robot_root: str) -> dict:
    """Return footprint size + Z-offset + inscribed/circumscribed radii."""
    import isaacsim.core.experimental.utils.bounds as bounds_utils
    import numpy as np
    from pxr import Usd, UsdPhysics

    import antioch

    stage = antioch.stage()
    collider_paths = [str(prim.GetPath()) for prim in Usd.PrimRange(stage.GetPrimAtPath(robot_root)) if prim.HasAPI(UsdPhysics.CollisionAPI)]
    if not collider_paths:
        collider_paths = [str(stage.GetPrimAtPath(robot_root).GetPath())]

    aabb = bounds_utils.compute_combined_aabb(collider_paths)  # [xmin, ymin, zmin, xmax, ymax, zmax]
    mn, mx = aabb[:3], aabb[3:]
    size = mx - mn

    origin_z = stage.GetPrimAtPath(robot_root).GetAttribute("xformOp:translate").Get()[2]
    z_offset = max(0.0, origin_z - mn[2])  # how far the origin sits above the lowest collider

    half_w, half_d = size[0] / 2.0, size[1] / 2.0
    return {
        "size": tuple(size),  # full footprint extents (m)
        "z_offset": float(z_offset),  # origin -> lowest collider (m)
        "inscribed_radius": float(min(half_w, half_d)),  # safe for ANY yaw
        "circumscribed_radius": float(np.hypot(half_w, half_d)),  # worst-case yaw
        "aabb_min": tuple(mn),
        "aabb_max": tuple(mx),
    }
```

- Use `inscribed_radius` when the robot can rotate in place
  (over-conservative, zero clipping).
- Use `circumscribed_radius` only when zero false negatives are required.
- For non-circular robots (quadrupeds, long wheelbase AMRs), prefer the
  oriented-footprint check below over any single radius.
- **Spawn at `z = ground + fp["z_offset"]`** — skipping the offset is the
  falls-through-the-floor / floats-above-ground bug.

### Reference values (sanity check only)

If your derived footprint is far from these, suspect the collider authoring
or the stage units — not the math:

| Robot | Expected size (m) | Expected z_offset | Inscribed r |
|---|---|---|---|
| Quadruped (~Spot class) | ~1.1 × 0.4 × 0.55 | ~0.69 | ~0.22 |
| Indoor AMR (~Nova Carter class) | wheel_r 0.14, track 0.5 | ~0.0 | ~0.25 |
| Outdoor 6-wheel AMR | ~2.5 × 1.7 | ~0.0 | ~0.86 |
| Tabletop diff-drive (~Jetbot class) | wheel_r 0.03, base 0.11 | ~0.02 | ~0.06 |
| Humanoid (~H1 class) | — | ~1.05 | ~0.20 |

## Buffer sizing — derived, not magic

```python
def compute_buffer_cells(fp: dict, resolution: float, safety_margin: float = 0.30) -> int:
    buffer_m = fp["circumscribed_radius"] + safety_margin
    return int(round(buffer_m / resolution))
```

| Context | Safety margin |
|---|---|
| Open corridor, smooth control | 0.10 m |
| Cluttered aisle | 0.30 m |
| Cluttered + non-zero yaw error | 0.50 m |

Legacy blanket values (e.g. 1.5 m for everything) were tuned against
circular proxies; the oriented-footprint check below recovers that wasted
navigable space.

## Path planning

Erode by the inscribed radius (fast, conservative), plan over the eroded
grid, then validate the smoothed path with the oriented-footprint check —
which is what recovers the space the erosion threw away.

```python
def erode_for_planning(grid: "np.ndarray", fp: dict, resolution: float) -> "np.ndarray":
    import numpy as np
    from scipy.ndimage import binary_erosion

    kernel_r = int(fp["inscribed_radius"] / resolution)
    yy, xx = np.mgrid[-kernel_r : kernel_r + 1, -kernel_r : kernel_r + 1]
    kernel = xx**2 + yy**2 <= kernel_r**2
    return binary_erosion(grid == 0, structure=kernel)
```

Planning over the eroded grid:
`isaacsim.replicator.experimental.mobility_gen.impl.path_planner.generate_paths(start, freespace)`
runs BFS from a start cell (no goal) over the freespace and returns a tree
— call `unroll_path(end)` on the result to reconstruct the route to any
reachable cell, which suits any-reachable-goal routing and SDG sampling.
For goal-directed routing use a plain heapq A* — 8-connected with octile
heuristic. Smooth the raw path with Catmull-Rom and assign each waypoint
`yaw = atan2(dy, dx)` along the curve.

## Oriented-footprint collision check (default for non-circular robots)

Two modes, same question: does the robot's rectangle at this pose hit
anything?

### Mode 1 — against the PhysX scene (after sim init)

Drives the same query the simulator uses. Returns hit count; > 0 means
clip.

```python
def footprint_clips(x: float, y: float, yaw: float, fp: dict, z_query: float = 0.2) -> bool:
    import carb
    import numpy as np
    from omni.physx import get_physx_scene_query_interface
    from pxr import Gf

    half = carb.Float3(fp["size"][0] / 2, fp["size"][1] / 2, fp["size"][2] / 2)
    origin = carb.Float3(x, y, z_query + fp["size"][2] / 2)
    rot = Gf.Rotation(Gf.Vec3d(0, 0, 1), np.degrees(yaw)).GetQuat()
    quat = carb.Float4(*rot.GetImaginary(), rot.GetReal())
    hits = get_physx_scene_query_interface().overlap_box(
        half,
        origin,
        quat,
        lambda h: True,
        anyHit=True,  # early-exit on first hit
    )
    return hits > 0
```

### Mode 2 — against the rasterized grid (no PhysX needed)

Stamp the rotated rectangle and AND with the occupied mask. Works pre-play,
which makes it the right tool for validating a path before the sim starts:

```python
def footprint_clips_grid(px: float, py: float, yaw: float, fp: dict, grid: "np.ndarray", resolution: float) -> bool:
    import cv2
    import numpy as np

    w_cells = fp["size"][0] / resolution
    h_cells = fp["size"][1] / resolution
    rect = ((px, py), (w_cells, h_cells), np.degrees(yaw))
    pts = cv2.boxPoints(rect).astype(np.int32)
    mask = np.zeros_like(grid, dtype=np.uint8)
    cv2.fillPoly(mask, [pts], 1)
    return bool(np.any((grid > 0) & (mask > 0)))
```

`cv2` ships with the Isaac Sim runtime environment; if a given engine
image lacks it, replace `boxPoints`/`fillPoly` with a numpy polygon raster
— the logic is identical.

## Validation pipeline

1. `compute_robot_footprint(robot_root)` — size, z_offset, radii.
2. Rasterize obstacles (0.10–0.25 m resolution typical).
3. Binary-erode with a circular kernel of `inscribed_radius / resolution`.
4. Plan on the eroded grid (BFS-with-unroll or heapq A*).
5. Catmull-Rom smooth; assign yaw along the curve.
6. **Run the oriented-footprint check at every smoothed waypoint.**
7. One failing waypoint → snap to the nearest navigable cell, re-validate.
   Multiple failing → re-plan with a `circumscribed_radius` erosion kernel.

Skipping steps 6–7 is the classic "looks fine on the map, clips the rack in
the run" bug — worst on rectangular robots cornering through aisles.

## Reading the robot pose at runtime

During simulation, read simulated state, not authored USD:

```python
def robot_pose(robot: "Articulation") -> tuple:
    pos_wp, quat_wp = robot.get_world_poses()
    pos = pos_wp.numpy()[0]  # (3,)
    quat = quat_wp.numpy()[0]  # (4,) [w, x, y, z]
    return pos, quat
```

`XformCache` reads the authored layer and returns the initial pose forever
once the sim is running — the XformCache trap, owned by `isaac-sim-6`.
