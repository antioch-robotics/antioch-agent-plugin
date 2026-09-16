# Advanced

Adapted from NVIDIA [`spatial-reasoning/advanced.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/advanced.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Bbox Offset Correction (Critical for Asset Placement)

Most USD assets are NOT centered at origin. When placing an asset at a target position,
you must subtract the asset's bbox center to land it where you want it:

```python
# Target position from cube block
target_x, target_y, target_z = block["x"], block["y"], block["z"]

# Asset bbox center (computed from BBoxCache)
asset_cx, asset_cy, asset_cz_min = bbox["cx"], bbox["cy"], bbox["cz"]

# Corrected translation
translate = Gf.Vec3d(
    target_x - asset_cx,  # X offset correction
    target_y - asset_cy,  # Y offset correction
    -asset_cz_min,  # Ground the asset (base at Z=0)
)
```

**Without this correction, assets land tens or hundreds of meters from their intended position.**

### Computing Bbox in Kit Runtime (Most Reliable)

```python
bc = UsdGeom.BBoxCache(Usd.TimeCode.Default(), [UsdGeom.Tokens.default_])
# Temporarily reference asset to get composed bbox
test = stage.DefinePrim("/BBoxTest", "Xform")
test.GetReferences().AddReference(asset_path)
for _ in range(5):
    app.update()  # composition resolve; check the bound is finite, not a fixed-frame proof
r = bc.ComputeWorldBound(test).ComputeAlignedRange()
mn, mx = r.GetMin(), r.GetMax()
stage.RemovePrim("/BBoxTest")
```

### Per-Block vs Tiled Placement

- **Per-block**: One asset per cube placeholder. Best for small assets (racks, conveyors). No overshoot.
- **Tiled**: One large assembly covers multiple cubes. Risk of corridor intrusion. Validate zone boundaries.
- **Rule of thumb**: If asset is >50% of zone width, use per-block placement.
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
    c, s = math.cos(angle_rad), math.sin(angle_rad)
    k = axis
    dot = sum(a * b for a, b in zip(k, v))
    cross = (k[1] * v[2] - k[2] * v[1], k[2] * v[0] - k[0] * v[2], k[0] * v[1] - k[1] * v[0])
    return tuple(v[i] * c + cross[i] * s + k[i] * dot * (1 - c) for i in range(3))
```

### Polar Decomposition (Extract Rotation + Scale from Matrix)

Given an arbitrary 4×4 matrix M, decompose into M = T·R·S:

_See `decompose_transform()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (26 lines)._


## Spatial Indexing for Large Scenes

### R-Tree (Best for 2D/3D range queries — warehouse floor plans)

R-trees group nearby objects into bounding rectangles, enabling O(log n) range queries instead of O(n).

_See `__init__()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (47 lines)._


**When to use which spatial index:**

| Structure | Best For | Query Time | Build Time |
|---|---|---|---|
| R-tree | Range queries, overlapping bboxes | O(log n) | O(n log n) |
| k-d tree | Nearest neighbor, point queries | O(log n) | O(n log n) |
| Octree | 3D scenes, LOD, frustum culling | O(log n) | O(n log n) |
| Grid | Uniform density, simple collision | O(1) | O(n) |

For warehouse layouts (1000-5000 prims, mostly 2D floor placement): **uniform grid** is fastest and simplest.

### Uniform Grid (Practical for Warehouse Collision Detection)

_See `__init__()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (35 lines)._


## Collision Detection

### Separating Axis Theorem (SAT) — OBB vs OBB

Two convex shapes don't overlap if and only if there exists an axis where their projections don't overlap. For two OBBs in 2D, test 4 axes (2 edge normals per box):

```python
def obb_overlap_2d(center_a, half_ext_a, angle_a, center_b, half_ext_b, angle_b):
    """Test if two 2D OBBs overlap using SAT."""

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

Raw A* paths are jagged. Smooth with Catmull-Rom interpolation:

_See `catmull_rom_point()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (36 lines)._


### Dubins Paths (Non-Holonomic Vehicles — Forklifts)

For vehicles that can't strafe (forklifts, AGVs), Dubins paths give the shortest path composed of arcs and straight lines:

```python
def dubins_path_length(start, goal, min_radius):
    """Compute Dubins path length between two poses (x, y, heading_rad).
    Returns length of shortest CSC or CCC path."""
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

_See `__init__()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (74 lines)._


## Visibility & Camera Mathematics

### Focal Length ↔ Field of View Conversion

```python
def focal_to_fov(focal_mm, sensor_width_mm=36.0):
    """Convert focal length to horizontal FOV (degrees). Default: 36mm full-frame."""
    return 2 * math.degrees(math.atan(sensor_width_mm / (2 * focal_mm)))


def fov_to_focal(fov_deg, sensor_width_mm=36.0):
    """Convert horizontal FOV to focal length (mm)."""
    return sensor_width_mm / (2 * math.tan(math.radians(fov_deg / 2)))


# Common warehouse camera settings:
# 12mm → 84.1° FOV — ultra-wide, top-down overview
# 14mm → 75.4° FOV — wide overview
# 18mm → 63.4° FOV — general purpose
# 24mm → 49.1° FOV — corridor view
# 35mm → 34.4° FOV — detail/equipment close-up
# 50mm → 24.4° FOV — tight detail
```

### Frustum Culling

_See `frustum_planes()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (35 lines)._


## Coordinate System Conversions

```python
# USD (OpenUSD): Z-up, right-handed, meters
# Unity: Y-up, left-handed, meters
# Unreal: Z-up, left-handed, centimeters


def usd_to_unity(x, y, z):
    """USD (Z-up, RH) → Unity (Y-up, LH): swap Y↔Z, negate X."""
    return (-x, z, y)


def unity_to_usd(x, y, z):
    """Unity (Y-up, LH) → USD (Z-up, RH): swap Y↔Z, negate X."""
    return (-x, z, y)


def usd_to_unreal(x, y, z):
    """USD (Z-up, RH, meters) → Unreal (Z-up, LH, cm): negate Y, scale ×100."""
    return (x * 100, -y * 100, z * 100)


def unreal_to_usd(x, y, z):
    """Unreal (Z-up, LH, cm) → USD (Z-up, RH, meters): negate Y, scale ÷100."""
    return (x / 100, -y / 100, z / 100)
```

## Warehouse Layout Standards (Real-World)

### Aisle Width Requirements

| Type | Min Width | Typical | Standard |
|---|---|---|---|
| VNA (Very Narrow Aisle) | 1.6m | 1.8m | EN 15620, ANSI/RMI MH16.1 |
| Conventional forklift | 3.0m | 3.5m | OSHA 1910.176 |
| Wide aisle (counterbalance) | 3.5m | 4.0-4.5m | OSHA 1910.178 |
| Pedestrian only | 0.9m | 1.2m | ADA, OSHA 1910.22 |
| Fire lane | 2.4m | 3.0m | NFPA 13 / local code |
| Truck dock approach | 18m | 25-30m | For 16m trailer turning |

### Rack Height & Stability

| Storage Type | Max Height | Clearance to Sprinkler | Standard |
|---|---|---|---|
| Standard selective | 7.6m (25ft) | 457mm (18in) min | NFPA 13, FM Global |
| Double-deep | 7.6m | 457mm | RMI |
| Drive-in/through | 10.7m (35ft) | 457mm | NFPA 13 |
| VNA/ASRS | 12-40m | 457mm min, 914mm preferred | EN 15512 |
| Mezzanine clearance | — | 2.3m min under, 2.1m above | OSHA 1910.22 |

### Loading Dock Dimensions

| Element | Dimension | Notes |
|---|---|---|
| Door opening width | 2.6m (8.5ft) | Standard trailer width |
| Door opening height | 3.0m (10ft) | Standard trailer height |
| Dock height | 1.2m (48in) | Standard trailer bed height |
| Dock leveler length | 1.8-3.0m | Bridges gap + height difference |
| Approach apron | 18-30m | Turning radius for 16m trailer |
| Dock seal/shelter | 3.0×3.4m | Weather protection |
| Dock spacing (center-center) | 3.6-4.0m | One per trailer bay |

### Fire Safety Spacing (NFPA 13)

| Configuration | Flue Space | Transverse | Longitudinal |
|---|---|---|---|
| Single-row rack | 76mm (3in) | — | — |
| Double-row rack | 152mm (6in) min | 152mm | 76mm |
| ASRS (high-pile) | As designed | — | — |
| Top-of-storage to sprinkler | 457mm (18in) min | — | — |
| In-rack sprinklers needed | >3.7m (12ft) high | — | NFPA 13 Ch. 20 |

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

**Golden zone:** Items picked most frequently should be at ergonomic height (waist to shoulder: 0.6-1.5m) and closest to shipping docks.

### Travel Distance Optimization

```python
def total_travel_distance(pick_list, rack_positions, dock_position):
    """Compute total travel distance for a pick route using nearest-neighbor."""
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
EPSILON = 1e-7  # For single-precision float comparisons
EPSILON_D = 1e-12  # For double-precision


def nearly_equal(a, b, eps=EPSILON):
    return abs(a - b) <= eps * max(1.0, abs(a), abs(b))


def bbox_valid(mn, mx, eps=1e30):
    """Check if a bbox is valid (not infinity/NaN)."""
    return all(mn[i] < eps and mx[i] > -eps and mn[i] <= mx[i] for i in range(3))


def safe_normalize(v, fallback=(1, 0, 0)):
    """Normalize vector with degeneracy handling."""
    length = math.sqrt(sum(c * c for c in v))
    if length < EPSILON:
        return fallback
    return tuple(c / length for c in v)
```

### Robust Orientation Test (Shewchuk-style)

```python
def orient2d(a, b, c):
    """Robust 2D orientation test. Returns:
    > 0 if c is left of line a→b (counterclockwise)
    < 0 if c is right (clockwise)
    = 0 if collinear
    Uses exact arithmetic expansion for robustness."""
    # Simple version (sufficient for warehouse-scale coordinates)
    det = (b[0] - a[0]) * (c[1] - a[1]) - (b[1] - a[1]) * (c[0] - a[0])
    return det
```

## Additional theoretical notes

### Dual Quaternion Skinning
Combines rotation and translation for smooth deformation. Dual quaternion: `q̂ = q + ε·t·q`
where `q` is rotation quaternion and `t` is translation. Interpolate: slerp the real part,
lerp the dual part. Use for skinned mesh animation (digital humans walking).

### OBB via PCA
Compute Oriented Bounding Box using Principal Component Analysis:
1. Compute covariance matrix of mesh vertices
2. Eigenvectors = OBB axes, eigenvalues = spread along each axis
3. Project vertices onto eigenvectors for tight extents
Result: Tighter fit than AABB, especially for elongated objects (conveyors, racks).

### Seismic Bracing Rule
`bracing_factor = rack_height / base_width`
- If factor > 2.0: seismic bracing required (cross-bracing between uprights)
- If factor > 3.0: engineering review mandatory
- Standards: EN 15512 for Europe; Rack Manufacturers Institute ANSI MH16.1 for the US

### Bin Packing Complexity
- 2D bin packing: NP-hard. First-Fit Decreasing Height (FFDH) achieves 1.7× optimal.
- 3D bin packing: NP-hard. Bottom-Left-Fill heuristic works for pallet loading.
- Strip packing (conveyors): 2× approximation ratio is achievable in polynomial time.
- MaxRects (Best Short Side Fit): Best practical heuristic for floor space allocation — 85-95% utilization typical.

### BSP Trees for PVS (Potentially Visible Set)
For large warehouse interiors (>1000 prims), precompute PVS using BSP (Binary Space Partition):
1. Subdivide space along major architectural planes (rack rows, corridor walls)
2. For each cell, ray-cast to determine which other cells are visible
3. Store as bitfield: `pvs[cell_id] = set(visible_cell_ids)`
4. At render time, only submit geometry from visible cells
Reduces draw calls from O(n) to O(visible) — critical for real-time warehouse flythrough.

### Dubins Path Types
Six path types for non-holonomic vehicles: LSL, RSR, LSR, RSL, RLR, LRL
- L = left turn (arc), R = right turn (arc), S = straight
- Minimum turning radius for warehouse AGVs: 1.5-2.0m
- For forklifts (Ackermann steering): 2.5-3.5m minimum radius
- Dubins path gives the shortest C¹-continuous path between two oriented poses

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
For warehouse containment, derive interior from shell bbox:
```python
shell_range = bbox_cache.ComputeWorldBound(shell_prim).ComputeAlignedRange()
MARGIN = 2.0  # meters clearance from walls
interior_min = shell_range.GetMin() + Gf.Vec3d(MARGIN, MARGIN, 0)
interior_max = shell_range.GetMax() - Gf.Vec3d(MARGIN, MARGIN, 0)
```

### Delta Transform Rule
When modifying transforms on prims with child geometry:
1. Read original matrix: `orig = xf.ComputeLocalToWorldTransform()`
2. Copy: `new_mat = Gf.Matrix4d(orig)`
3. Modify ONLY translation: `new_mat.SetTranslateOnly(old_t + delta)`
4. Write back: `xf.MakeMatrixXform().Set(new_mat)`

NEVER construct a fresh identity matrix and set a new position — this destroys
the rotation/scale that child prims depend on.

### Mandatory Post-Transform Verification
After modifying any prim transform, verify world bounds:
```python
for prim in modified:
    bb = bbox_cache.ComputeWorldBound(prim).ComputeAlignedRange()
    assert bb.GetMin()[0] >= IX_MIN, f"{prim.GetPath()} exceeds west wall"
    assert bb.GetMax()[0] <= IX_MAX, f"{prim.GetPath()} exceeds east wall"
    # ... all 4 walls
```
Zero AABB-containment violations required before treating the layout as inside the shell. This is geometric containment, not PhysX contact.

## Block proxy → real asset swap

### Solid cubes vs real geometry density gap
UsdGeom.Cube fills 100% of its footprint as a solid opaque slab.
Real rack compositions are 60-70% empty air (shelves + gaps between goods).
1:1 block replacement will ALWAYS look 40-50% sparser at top-down scale.

**Mitigation:**
- Add zone-aware floor clutter (pallets, boxes, crates) in every gap
- Target 0.3-0.8 items/m² depending on zone type
- Stack items 1-4 levels vertically for pallet staging areas
- Grid-based occupancy check prevents double-placement

### Asset scale matching
Each block has exact W×D×H dimensions. Real assets have different native sizes.
Must compute per-axis scale factors:
```python
sx = block_w / asset_w
sy = block_d / asset_d
sz = block_h / asset_h
# Clamp to prevent absurd distortion
sx, sy, sz = [max(0.05, min(s, 30.0)) for s in [sx, sy, sz]]
# Offset for asset origin
tx = block_x - asset_min_x * sx
ty = block_y - asset_min_y * sy
tz = block_z - asset_min_z * sz
mat = Gf.Matrix4d(sx, 0, 0, 0, 0, sy, 0, 0, 0, 0, sz, 0, tx, ty, tz, 1)
```
Without this, assets land at wrong positions and wrong sizes.

### Block dimensions are individual positions
Placeholder cubes are often individual rack/pallet positions (example 2–4 m wide, 1–2 m deep), not full rack rows. A long rack assembly (example 3.1×83.1 m) is far bigger than any single block. Place one asset per block at matched scale, not one giant assembly covering the row.

### Loaded vs empty rack variants
- Composition / loaded variants include crates/boxes on shelves.
- Base / empty-frame variants are frames only.
- Inspect the variant, do not assume a name means "loaded".

A vision-model density score is not placement evidence. Compare proxy vs real at the same camera, check zone type/size, and confirm bounds against the shell.

### Zone-aware clutter fill (example densities)
```python
ZONES = [
    # (name, x1, y1, x2, y2, density_per_m2, asset_pool, max_stack) — example warehouse
    ("Receiving", 3, 0, 170, 15, 0.6, pallets + boxes, 3),
    ("Storage_floor", 3, 16, 170, 90, 0.3, boxes, 2),
    ("Staging", 82, 90, 170, 100, 0.6, pallets + crates, 3),
    ("Shipping", 3, 125, 170, 134, 0.6, pallets + boxes, 3),
]
# Grid-walk at step = 1/sqrt(density), 85% fill rate, occupancy check
```

### Capture notes
- Viewport capture: `omni.kit.viewport.utility.capture_viewport_to_file` (or `antioch.capture_viewport()`). This is the active viewport, not a sensor camera or Replicator product. See [rendering.md](isaac-sim-rendering.md) / [sensors.md](isaac-sim-sensor.md).
- Settings: `carb.settings.get_settings()`, not `ExtensionManager.get_settings()`.
- Project Python must keep `pxr` / `omni` / `carb` / `isaacsim` imports inside functions or `TYPE_CHECKING`. Native fragments in this file are function/cell bodies after startup.
- Replicator annotators can fail on very heavy stages; viewport capture is a fallback for a picture, not a substitute for required annotators.
- `instanceable=True` with many unique composition meshes can exhaust GPU memory. Instance proxies are read-only.
- Add explicit lights when you want them; a high-intensity DomeLight can wash the background white. Default viewport lighting is a different look from a authored DomeLight.
- Malformed `geomSubset`s on some catalog assets: fall back to a simpler proxy or a known-good tile after inspecting the USD.


## Camera framing for full bounding box capture

When rendering a single object (e.g. stacked pallet), frame the camera to capture the entire bounding box:

_See `compute_camera_distance()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (31 lines)._


Key lessons:
- Eye height at object midpoint for balanced framing
- 3/4 view angle (45° from front) shows depth best
- Side views need longer focal length (50mm+) to reduce distortion
- Top-down: place camera at ~1.5× object height, target at object center
- Always add 20-30% margin to prevent clipping at edges

## Multi-object camera framing

When framing multiple objects (e.g., 5 pallets in a line):
```python
# Compute required distance from line length and FOV
line_len = last_y - first_y + object_width  # total span
focal = 22.0  # wide angle for groups
fov_h = 2 * math.atan(aperture / (2 * focal))
dist = (line_len / 2) / math.tan(fov_h / 2) * 1.3  # 30% margin

# Place camera at 35° orbit, 25° elevation from center of group
center_y = (first_y + last_y) / 2
eye = (dist * cos(elev) * sin(angle), center_y - dist * cos(elev) * cos(angle), center_z + dist * sin(elev))
```

## Placement with PointInstancers

For repeated identical objects (boxes on pallets, rack bays, etc.), use `UsdGeom.PointInstancer` instead of individual prims. Group by (asset, rotation) — each group gets one prototype and an array of positions.

**Key spatial reasoning for instanced placement:**
- Positions are in WORLD SPACE (not local to instancer)
- Rotation baked into prototype means all instances share the same orientation
- For cross-stacking, create SEPARATE instancers for 0° and 90° groups
- Grid math is identical whether using prims or instancers — the stacker computes positions the same way
- A point instancer is not an independently simulated articulation

## Surface detection on mesh geometry

When placing objects on top of mesh geometry (e.g., pallets on rack beams), vertex positions alone are insufficient. Three approaches, in order of accuracy:

### 1. Vertex Mean Clustering (least accurate)
```python
# Cluster vertex Z values, use mean of cluster
# Problem: beam meshes extend above the mean → objects penetrate
# Typical error: ~8mm below true surface
```

### 2. Vertex z_max with Normal Filtering (good)
```python
# Only use vertices from upward-facing faces (normal.z > 0.7)
# Use z_max of each cluster
# Typical error: ~2-3mm (add 5mm clearance)
normals = mesh.GetNormalsAttr().Get()
for face in faces:
    if face_normal.z > 0.7:
        surface_z = max(vertex.z for vertex in face_verts)
```

### 3. PhysX Raycast (most accurate, TODO)
```python
# Cast ray downward from above, get exact hit point
# Bypasses all vertex analysis — gets renderer's actual surface
# from omni.physx import get_physx_scene_query_interface
# Requires collision mesh on the rack
```

**Key lesson:** Thin horizontal geometry (beams, shelves, panels) has vertex distributions that don't cluster cleanly. The "top surface" is a subset of vertices on upward-facing faces, not the cluster center or even cluster max of ALL vertices. Always filter by face normal direction before computing placement Z.

Antioch owns the remote Kit process. Do not kill, restart, or drive Isaac through host JSON command files. Reuse the session for shots that share scene state; see [../SKILL.md](../SKILL.md) and [antioch-platform](../../antioch-platform/SKILL.md).
