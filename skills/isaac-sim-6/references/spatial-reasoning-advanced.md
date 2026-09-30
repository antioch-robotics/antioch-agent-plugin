# Advanced spatial reasoning

Adapted from NVIDIA [`spatial-reasoning/advanced.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/advanced.md) (Apache-2.0).

[Isaac Sim task index](../SKILL.md#domain-references) · [Antioch startup](../../antioch-platform/references/simulation-code.md)

Use this after [units and transforms](spatial-reasoning.md) for large layouts, tight placement, paths, or repeated assets. The linked upstream scripts are examples to inspect and adapt, not installed helpers. Run native fragments inside a function or cell after [startup](../../antioch-platform/references/simulation-code.md).

## Bbox offset correction

A USD asset may have an origin far from its geometry. When placing an asset at a target position,
align its transformed bounds with the target. This example assumes an unrotated asset already in the parent's units:

```python
# Target position from cube block
target_x, target_y, target_z = block["x"], block["y"], block["z"]

# Asset bbox center (computed from BBoxCache)
asset_cx, asset_cy, asset_cz_min = bbox["cx"], bbox["cy"], bbox["min_z"]

# Corrected translation
translate = Gf.Vec3d(
    target_x - asset_cx,  # X offset correction
    target_y - asset_cy,  # Y offset correction
    target_z - asset_cz_min,  # Align the base with the support height
)
```

For rotated or scaled assets, transform the bounds before choosing the support point. Author the offset on a placement wrapper so the referenced asset keeps its original transform stack. See [the asset pipeline](usd-pipeline.md#the-placeholder-to-asset-pipeline).

### Computing composed bounds

```python
from pxr import Usd, UsdGeom

bc = UsdGeom.BBoxCache(Usd.TimeCode.Default(), [UsdGeom.Tokens.default_])
# Use a temporary path owned by this measurement.
test = stage.DefinePrim("/BBoxTest", "Xform")
test.GetReferences().AddReference(asset_path)
r = bc.ComputeWorldBound(test).ComputeAlignedRange()
mn, mx = r.GetMin(), r.GetMax()
# Check finite, nonempty bounds; missing payloads or references need investigation.
stage.RemovePrim("/BBoxTest")
```

### Per-Block vs Tiled Placement

- **Per-block**: One asset per cube placeholder. Best for small assets (racks, conveyors). No overshoot.
- **Tiled**: One large assembly covers multiple cubes. Risk of corridor intrusion. Validate zone boundaries.
- Measure the assembly against the available zone and clearances before choosing either strategy.

## Advanced Transform Mathematics

### Quaternion Rotation (Gimbal Lock Avoidance)

Euler angles (XYZ rotation) suffer from gimbal lock when pitch approaches ±90°. Quaternions avoid this.

**Quaternion representation:** `q = w + xi + yj + zk` where `w² + x² + y² + z² = 1`

_See `euler_to_quat()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (38 lines)._


**When to use quaternions in USD:**
- Smooth camera interpolation between keyframes (slerp)
- Avoiding gimbal lock in robot joint animations
- Decomposing arbitrary transform matrices

### Rodrigues Rotation Formula

Rotate vector `v` by angle `θ` around unit axis `k`:

```
v_rot = v·cos(θ) + (k × v)·sin(θ) + k·(k · v)·(1 - cos(θ))
```

```python
def rodrigues_rotate(v, axis, angle_rad):
    """Rotate vector v around unit axis by angle (radians)."""
    import math

    c, s = math.cos(angle_rad), math.sin(angle_rad)
    length = math.sqrt(sum(value * value for value in axis))
    if length == 0:
        raise ValueError("Rotation axis must be nonzero")
    k = tuple(value / length for value in axis)
    dot = sum(a * b for a, b in zip(k, v))
    cross = (k[1] * v[2] - k[2] * v[1], k[2] * v[0] - k[0] * v[2], k[0] * v[1] - k[1] * v[0])
    return tuple(v[i] * c + cross[i] * s + k[i] * dot * (1 - c) for i in range(3))
```

### Polar Decomposition (Extract Rotation + Scale from Matrix)

For a Gf row-vector transform without shear, scale, rotation, and translation compose as `S * R * T`. A general matrix can also contain shear or reflection; do not silently discard either when decomposing it:

_See `decompose_transform()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (26 lines)._


## Spatial Indexing for Large Scenes

### R-Tree (Best for 2D/3D range queries — warehouse floor plans)

R-trees group nearby objects into bounding rectangles so a query can skip unrelated regions. Cost depends on overlap and on how many results the query returns.

_See `RTreeNode` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (47 lines)._


**When to use which spatial index:**

| Structure | Useful for | Tradeoff |
|---|---|---|
| R-tree | Range queries and overlapping bounds | Overlapping nodes and many results increase query work |
| k-d tree | Nearest-neighbor and point queries | Rebuild or update when points move; high dimensions reduce pruning |
| Octree | Hierarchical 3D occupancy and spatial queries | Depth and sparse cells affect storage and traversal |
| Grid | Nearby candidates at roughly uniform density | Cell size controls bucket size and the number of visited cells |

A grid is a useful first implementation for floor layouts. Profile the actual scene before choosing an index; update affected entries when objects move.

### Uniform Grid (Practical for Warehouse Collision Detection)

_See `SpatialGrid` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (35 lines)._


## Collision Detection

### Separating Axis Theorem (SAT) — OBB vs OBB

Two convex shapes don't overlap if and only if there exists an axis where their projections don't overlap. For two OBBs in 2D, test 4 axes (2 edge normals per box):

```python
def obb_overlap_2d(center_a, half_ext_a, angle_a, center_b, half_ext_b, angle_b):
    """Test if two 2D OBBs overlap using SAT."""

    import math

    def get_axes(angle):
        c, s = math.cos(angle), math.sin(angle)
        return [(c, s), (-s, c)]

    def project(center, half_ext, angle, axis):
        axes = get_axes(angle)
        corners = []
        for sx in (-1, 1):
            for sy in (-1, 1):
                cx = center[0] + sx * half_ext[0] * axes[0][0] + sy * half_ext[1] * axes[1][0]
                cy = center[1] + sx * half_ext[0] * axes[0][1] + sy * half_ext[1] * axes[1][1]
                corners.append(cx * axis[0] + cy * axis[1])
        return min(corners), max(corners)

    for angle in (angle_a, angle_b):
        for axis in get_axes(angle):
            min_a, max_a = project(center_a, half_ext_a, angle_a, axis)
            min_b, max_b = project(center_b, half_ext_b, angle_b, axis)
            if max_a < min_b or max_b < min_a:
                return False  # Separating axis found — no collision
    return True  # All axes overlap — collision
```

### GJK Algorithm (Simplified for Convex Polygons)

The Gilbert-Johnson-Keerthi algorithm determines if two convex shapes overlap by searching for the origin in their Minkowski difference:

_See `gjk_overlap_2d()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (43 lines)._


## Path Planning Algorithms

### A* for Warehouse Grid Navigation

_See `astar_warehouse()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (48 lines)._


### Path Smoothing (Cubic Catmull-Rom Spline)

Raw A* paths are jagged. Catmull-Rom interpolation can smooth them, but can also leave the free corridor. Recheck the swept robot footprint and turning limits after smoothing:

_See `catmull_rom_point()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (36 lines)._


### Dubins Paths (Non-Holonomic Vehicles — Forklifts)

For an ideal vehicle that moves forward with bounded curvature, Dubins paths combine arcs and straight lines. Compare all six feasible path families to find the shortest. The upstream sketch below computes only the LSL candidate; it is not a full planner or an obstacle check:

```python
def dubins_lsl_length(start, goal, min_radius):
    """Length of the LSL candidate between poses (x, y, heading_rad)."""
    import math

    if min_radius <= 0:
        raise ValueError("Turning radius must be positive")
    dx = goal[0] - start[0]
    dy = goal[1] - start[1]
    d = math.sqrt(dx * dx + dy * dy) / min_radius
    theta = math.atan2(dy, dx)
    alpha = (start[2] - theta) % (2 * math.pi)
    beta = (goal[2] - theta) % (2 * math.pi)

    # LSL path (Left-Straight-Left) — one of 6 Dubins path types
    p_sq = 2 + d * d - 2 * math.cos(alpha - beta) + 2 * d * (math.sin(alpha) - math.sin(beta))
    if p_sq < 0:
        return float("inf")
    p = math.sqrt(p_sq)
    tmp = math.atan2(math.cos(beta) - math.cos(alpha), d + math.sin(alpha) - math.sin(beta))
    t = (-alpha + tmp) % (2 * math.pi)
    q = (beta - tmp) % (2 * math.pi)
    return (t + p + q) * min_radius
```

## Packing Algorithms

### 2D Maximal Rectangles Bin Packing (Floor Space Allocation)

_See `MaxRectsBinPack` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (74 lines)._


## Visibility & Camera Mathematics

### Focal Length ↔ Field of View Conversion

```python
import math


def focal_to_fov(focal_mm, sensor_width_mm=36.0):
    """Convert focal length to horizontal FOV (degrees). Default: 36mm full-frame."""
    return 2 * math.degrees(math.atan(sensor_width_mm / (2 * focal_mm)))


def fov_to_focal(fov_deg, sensor_width_mm=36.0):
    """Convert horizontal FOV to focal length (mm)."""
    return sensor_width_mm / (2 * math.tan(math.radians(fov_deg / 2)))


# With a 36 mm aperture: 18 mm focal length gives 90 degrees horizontal FOV.
# Use the camera's actual aperture; focal length alone does not define FOV.
```

USD cameras default to a horizontal aperture of 20.955 mm. The helpers above deliberately use a 36 mm full-frame example; pass the actual camera aperture for framing.

### Frustum Culling

_See `frustum_planes()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (35 lines)._


## Coordinate System Conversions

Read each asset's units, up-axis, handedness, and forward axis. USD permits different up-axes and unit scales. A change of basis must apply to positions, directions, rotations, and normals consistently; normals need an inverse-transpose under nonuniform scaling.

For example, one convention mapping Z-up right-handed coordinates to Y-up left-handed coordinates swaps Y and Z: `(x, y, z) -> (x, z, y)`. This reverses handedness. Negating another axis would reverse it again. Decide where forward should point before choosing the mapping. For orientation matrices, use the corresponding basis transformation rather than swapping quaternion components by intuition.

For a Z-up, right-handed stage measured in meters, an Unreal mapping that keeps +X forward is `(x, y, z) -> (100*x, -100*y, 100*z)`. The inverse is `(x/100, -y/100, z/100)`. This converts to left-handed centimeters; confirm the source forward axis before applying it.

## Warehouse layout constraints

The [upstream warehouse tables](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/advanced.md#warehouse-layout-standards-real-world) include sizing values and standards citations that have not been verified here. Treat a simulation layout as task input, not a certified facility design. Derive the following from the actual robot, asset dimensions, facility plans, and applicable requirements:

| Constraint | Measure or supply |
|---|---|
| Travel aisles | Swept robot footprint, turning radius, passing space, localization margin |
| Rack placement | Assembly bounds, payload envelope, service access, floor and support constraints |
| Loading docks | Vehicle approach, door opening, dock height, clearance for loading equipment |
| Pedestrian and emergency routes | The facility's required clearance and access rules |
| Sprinklers and rack bracing | Approved facility requirements; a bounding-box ratio cannot establish compliance |

Keep these constraints in layout parameters so the same checks run after every placement change. Do not infer walkable space from a shell's outer bounding box.

### ABC Analysis for SKU Placement

```python
def abc_classify(skus):
    """Classify SKUs into A/B/C based on cumulative revenue/picks.
    A = top 20% of items → 80% of picks → closest to shipping
    B = next 30% → 15% of picks → middle zone
    C = bottom 50% → 5% of picks → furthest zone
    """
    sorted_skus = sorted(skus, key=lambda s: s["annual_picks"], reverse=True)
    total_picks = sum(s["annual_picks"] for s in sorted_skus)
    if total_picks <= 0:
        return sorted_skus
    cumulative = 0
    for sku in sorted_skus:
        cumulative += sku["annual_picks"]
        ratio = cumulative / total_picks
        if ratio <= 0.80:
            sku["class"] = "A"
        elif ratio <= 0.95:
            sku["class"] = "B"
        else:
            sku["class"] = "C"
    return sorted_skus
```

ABC placement groups items by demand. The 80% and 95% cutoffs above are example inputs; choose them for the workload. Place frequently picked items where the intended robot or operator can reach them, and measure travel distance rather than assuming a layout is efficient.

### Travel Distance Optimization

```python
def total_travel_distance(pick_list, rack_positions, dock_position):
    """Compute total travel distance for a pick route using nearest-neighbor."""
    import math

    pos = dock_position
    total = 0
    remaining = list(pick_list)
    while remaining:
        nearest = min(remaining, key=lambda p: math.sqrt((rack_positions[p][0] - pos[0]) ** 2 + (rack_positions[p][1] - pos[1]) ** 2))
        total += math.sqrt((rack_positions[nearest][0] - pos[0]) ** 2 + (rack_positions[nearest][1] - pos[1]) ** 2)
        pos = rack_positions[nearest]
        remaining.remove(nearest)
    total += math.sqrt((dock_position[0] - pos[0]) ** 2 + (dock_position[1] - pos[1]) ** 2)
    return total
```

## Numerical Stability

### Epsilon Comparisons

```python
import math

EPSILON = 1e-7  # Example tolerance; choose it for the scene scale and operation.


def nearly_equal(a, b, eps=EPSILON):
    return abs(a - b) <= eps * max(1.0, abs(a), abs(b))


def bbox_valid(mn, mx):
    """Check if a bbox is valid (not infinity/NaN)."""
    return all(math.isfinite(mn[i]) and math.isfinite(mx[i]) and mn[i] <= mx[i] for i in range(3))


def safe_normalize(v, fallback=(1, 0, 0)):
    """Normalize vector with degeneracy handling."""
    length = math.sqrt(sum(c * c for c in v))
    if length < EPSILON:
        return fallback
    return tuple(c / length for c in v)
```

### Orientation test

```python
def orient2d(a, b, c):
    """Floating-point 2D orientation test. Returns:
    > 0 if c is left of line a→b (counterclockwise)
    < 0 if c is right (clockwise)
    = 0 if collinear
    Near zero, floating-point rounding can change the sign."""
    # Use an adaptive exact predicate when near-collinear signs matter.
    det = (b[0] - a[0]) * (c[1] - a[1]) - (b[1] - a[1]) * (c[0] - a[0])
    return det
```

## Additional theoretical notes

### Dual Quaternion Skinning

Dual quaternions represent rigid rotation and translation together. They are useful for skeletal animation because blending rigid transforms avoids some matrix-blending artifacts. Use a library that normalizes the result and handles antipodal quaternion signs. This animation representation does not simulate physical contact.

### OBB via PCA

Compute the covariance of mesh vertices, find its eigenvectors, then project vertices on those axes to get oriented extents. This often gives tighter bounds for elongated objects than an AABB. It is not a minimum-volume box, and nearly symmetric geometry can produce unstable axes.

### Bin Packing

MaxRects and first-fit heuristics are useful ways to assign floor space. Score the result against the actual footprint, rotation restrictions, clearance, and access constraints. High area utilization can leave unusable aisles. See the upstream `MaxRects` example in the linked spatial helpers for a starting implementation.

### BSP Trees and Potentially Visible Sets

A large interior can be partitioned along walls or other architectural planes. A potentially visible set records regions that might be visible from each region, allowing a renderer to skip unrelated geometry. The set must be conservative: a few rays do not prove a region is invisible. Use the renderer's culling support and profile before implementing custom visibility logic.

### Dubins Path Types

The six path families are LSL, RSR, LSR, RSL, RLR, and LRL: L/R are constant-radius turns and S is straight motion. The model assumes forward-only motion and a minimum turning radius. Vehicles that reverse need another model, such as Reeds-Shepp. Derive turning limits from the actual robot and check obstacles separately; see [navigation primitives](navigation-primitives.md).

## Containment verification

### The origin ≠ geometry problem
A prim's transform origin can be very far from its geometry center.
Example: a rack assembly whose origin sits at X≈124 can have geometry spanning X=84→167 (83 m).
Checking if the origin is inside the container misses tens of meters of geometry.

**ALWAYS use BBoxCache for containment checks:**
```python
bbox_cache = UsdGeom.BBoxCache(Usd.TimeCode.Default(), [UsdGeom.Tokens.default_])
bb = bbox_cache.ComputeWorldBound(prim)
r = bb.ComputeAlignedRange()
world_min, world_max = r.GetMin(), r.GetMax()
# Check THESE against container bounds — not the transform origin
```

### Container Interior Bounds

Read the actual floor boundary, wall collision geometry, openings, and task zones. An outer shell AABB includes wall thickness, exterior space, and possibly empty courtyards. Shrinking it by a fixed margin does not recover the usable interior.

### Delta Transform Rule

Keep the prim's existing rotation and scale. A local matrix lives in parent space, so transform a world displacement by the inverse parent transform's linear part before changing local translation. `SetTranslateOnly()` preserves other matrix entries. Write to an owned placement wrapper or the appropriate existing translation op; replacing an imported animated stack can erase authored behavior. Never write a world matrix back as a local matrix under a transformed parent.

### Mandatory Post-Transform Verification
After modifying any prim transform, verify world bounds:
```python
bbox_cache.Clear()
for prim in modified:
    bb = bbox_cache.ComputeWorldBound(prim).ComputeAlignedRange()
    assert bb.GetMin()[0] >= IX_MIN, f"{prim.GetPath()} exceeds west wall"
    assert bb.GetMax()[0] <= IX_MAX, f"{prim.GetPath()} exceeds east wall"
    # ... all 4 walls
```
Zero AABB-containment violations required before treating the layout as inside the shell. This is geometric containment, not PhysX contact.

## Block proxy → real asset swap

### Solid cubes vs real geometry density gap

A cube fills its volume; a rack contains shelves, openings, and sometimes cargo. Compare the proxy and real scene from the same camera. Choose loaded or empty variants to match the task. Add pallets, boxes, or clutter only where the scenario needs them, and check access and overlap after placement.

### Asset scale matching

Measure native dimensions before replacing proxies. Prefer a module that fits naturally. If rescaling is part of the task, `sx = target_width / asset_width` gives an axis scale; apply the same reasoning to depth and height. Reject zero dimensions and inspect nonuniform scale because it changes mechanical dimensions, collision, mass, and inertia. Do not silently clamp a bad fit into an arbitrary range.

For an axis-aligned asset whose bounds are already in the wrapper's parent space, this matrix maps its minimum corner and dimensions to a target box:

```python
from pxr import Gf

size = asset_max - asset_min
if any(size[i] <= 0 or target_size[i] <= 0 for i in range(3)):
    raise ValueError("Asset and target dimensions must be positive")
scale = Gf.Vec3d(*(target_size[i] / size[i] for i in range(3)))
offset = Gf.Vec3d(*(target_min[i] - asset_min[i] * scale[i] for i in range(3)))
placement = Gf.Matrix4d().SetScale(scale) * Gf.Matrix4d().SetTranslate(offset)
# Apply to a new wrapper, not the asset's existing transform stack.
```

This example has no rotation. For a rotated target, transform and check all bound corners after placement.

### Block dimensions are individual positions

A placeholder may represent one rack bay rather than a complete row. Measure both the placeholder and the referenced assembly. Use one suitable module per position or tile an assembly across several positions, then verify zone boundaries and aisles.

### Loaded vs empty rack variants
- Composition / loaded variants include crates/boxes on shelves.
- Base / empty-frame variants are frames only.
- Inspect the variant, do not assume a name means "loaded".

A vision-model density score is not placement evidence. Compare proxy vs real at the same camera, check zone type/size, and confirm bounds against the shell.

### Zone-aware clutter fill

For a requested clutter density `d` objects per square meter, a grid spacing of `1 / sqrt(d)` is a starting point. Sample only within allowed zones, reject placements whose full bounds intersect obstacles, preserve travel corridors, and seed any randomness. Stacking also needs support-surface and contact checks. Density alone does not define a realistic warehouse.

### Capture notes
- Viewport capture: `omni.kit.viewport.utility.capture_viewport_to_file` (or `antioch.capture_viewport()`). This is the active viewport, not a sensor camera or Replicator product. See [rendering](isaac-sim-rendering.md) / [sensors](isaac-sim-sensor.md).
- Settings: `carb.settings.get_settings()`, not `ExtensionManager.get_settings()`.
- Project Python must keep `pxr` / `omni` / `carb` / `isaacsim` imports inside functions or `TYPE_CHECKING`. Native fragments in this file are function/cell bodies after startup.
- Replicator annotators can fail on very heavy stages; viewport capture is a fallback for a picture, not a substitute for required annotators.
- `instanceable=True` with many unique composition meshes can exhaust GPU memory. Instance proxies are read-only.
- Add explicit lights when you want them; a high-intensity DomeLight can wash the background white. Default viewport lighting is a different look from an authored DomeLight.
- Malformed `geomSubset`s on some catalog assets: fall back to a simpler proxy or a known-good tile after inspecting the USD.


## Camera framing for full bounding box capture

When rendering a single object (e.g. stacked pallet), frame the camera to capture the entire bounding box:

_See `compute_camera_distance()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (31 lines)._


Use the camera's horizontal and vertical FOV and all transformed bound corners. In the simple centered plane case, the required distance is `half_extent / tan(fov / 2)` on each axis; choose the larger result and add room for depth and a framing margin. Check the near/far clipping planes. Eye height at the object midpoint and a three-quarter view are useful starting points, not required views.

## Multi-object camera framing

For a group of objects, use their combined world bounds and both camera apertures. A bounding sphere gives a conservative starting distance that works at any orbit angle:

```python
def group_camera_eye(bounds_min, bounds_max, focal_mm, aperture_x_mm, aperture_y_mm, azimuth_deg=35, elevation_deg=25, margin=1.15):
    import math

    if min(focal_mm, aperture_x_mm, aperture_y_mm) <= 0 or margin < 1:
        raise ValueError("Lens dimensions must be positive and margin at least 1")
    center = tuple((lo + hi) / 2 for lo, hi in zip(bounds_min, bounds_max))
    radius = math.sqrt(sum((hi - lo) ** 2 for lo, hi in zip(bounds_min, bounds_max))) / 2
    if radius <= 0 or any(hi < lo for lo, hi in zip(bounds_min, bounds_max)):
        raise ValueError("Group bounds must have positive extent")
    half_fov = min(math.atan(aperture_x_mm / (2 * focal_mm)), math.atan(aperture_y_mm / (2 * focal_mm)))
    distance = radius / math.sin(half_fov) * margin
    azimuth, elevation = map(math.radians, (azimuth_deg, elevation_deg))
    eye = (
        center[0] + distance * math.cos(elevation) * math.sin(azimuth),
        center[1] - distance * math.cos(elevation) * math.cos(azimuth),
        center[2] + distance * math.sin(elevation),
    )
    return eye, center
```

Aim at the returned center using [look-at camera math](spatial-reasoning.md#look-at-camera-math). Read the actual focal length and horizontal/vertical aperture in matching units. Check clipping planes and inspect the render; a long row may look better with tighter framing than its bounding sphere provides.

## Placement with PointInstancers

For repeated geometry, `UsdGeom.PointInstancer` stores prototypes and per-instance transforms without one composed prim per copy. Group by the geometry and material requirements; orientations and scales can differ per instance.

Positions are **local to the instancer**, as specified by the [OpenUSD API](https://openusd.org/release/api/class_usd_geom_point_instancer.html). The instancer's transform maps them into the scene. Use `protoIndices`, `positions`, and optional orientations/scales with consistent array lengths. Separate instancers for 0° and 90° rotations are not required. A point instancer is not a group of independently controlled articulations.

## Surface detection on mesh geometry

When placing objects on top of mesh geometry (e.g., pallets on rack beams), vertex positions alone are insufficient. Three approaches, in order of accuracy:

### 1. Vertex clustering

Cluster vertex heights to find candidate shelves. Cluster means can fall inside the geometry, and maxima can include a rail rather than the support surface. Treat this as a coarse search.

### 2. Upward-facing surfaces

Inspect faces whose normals point toward the stage's up-axis, then find the support region under the object's whole footprint. Transform vertices and normals into a common space. Respect the mesh's normal interpolation and orientation; a single vertex height cannot establish stable support.

### 3. Physics scene query

A downward ray or shape query against the intended collision geometry can provide a support hit. Use the [physics query guidance](physics-simulation.md#raycast-scene-query) for the selected backend. Check units, filters, and collision approximation: the collision surface can differ from the rendered mesh. Multiple footprint samples or a shape query can catch gaps that a center ray misses.

These methods answer different questions. Verify the final placement and, when contact matters, let the physics settle and read live state before recording a [scenario check](../../scenario-design/SKILL.md).

Antioch owns the remote Kit process. Do not kill, restart, or drive Isaac through host JSON command files. Reuse the session for shots that share scene state; see [../SKILL.md](../SKILL.md) and [antioch-platform](../../antioch-platform/SKILL.md).
