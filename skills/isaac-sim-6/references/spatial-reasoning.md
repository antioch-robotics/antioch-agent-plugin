# 3D Spatial Reasoning

Adapted from NVIDIA [`spatial-reasoning/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/SKILL.md) (Apache-2.0).

[Isaac Sim task index](../SKILL.md#domain-references) · [Antioch startup](../../antioch-platform/references/simulation-code.md)

Geometric overlap of AABBs is not physical contact. A layout that looks clear in a top-down render is not a collision-free claim.

Use fragments below inside a function or notebook cell after startup.

See also [usd-composition-architecture.md](usd-composition-architecture.md), [usd-pipeline.md](usd-pipeline.md), and [spatial-reasoning-advanced.md](spatial-reasoning-advanced.md).

## Upstream helpers (source examples)

| Script | Purpose |
|---|---|
| [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) | Spatial helpers (look-at, bake_waypoints, GJK, A*, bbox) |

Not a bundled command.

## Coordinate system

- Read stage up-axis and `metersPerUnit`. Z-up / meters is a common convention, not a USD requirement.
- Placement coordinates are in stage units after conversion.
- Prefer a single authored rotation op (`AddRotateXYZOp()` or a matrix xform). Mixing leftover axis ops with a new matrix is a common source of double transforms.

## metersPerUnit conversion

Assets have their own metersPerUnit. Common values:
- `1.0` = meters (many robots, environments)
- `0.01` = centimeters (some catalog/composition assets)

USD references do not convert units or axes. For an unscaled asset: `scale = asset_meters_per_unit / stage_meters_per_unit`. Apply conversion once.

**Rule:** Before referencing any asset, read its metersPerUnit:
```python
asset_stage = Usd.Stage.Open(asset_path)
if not asset_stage:
    raise ValueError(f"Could not open {asset_path}")
mpu = UsdGeom.GetStageMetersPerUnit(asset_stage)
```
`Sdf.Layer.FindOrOpen()` can read binary crate layers. Use `Usd.Stage.Open()` here because bounds need the composed stage, including references and payloads, rather than one layer alone.

**Getting real-world size of an asset:**
```python
bbox_cache = UsdGeom.BBoxCache(Usd.TimeCode.Default(), [UsdGeom.Tokens.default_])
raw_range = bbox_cache.ComputeWorldBound(default_prim).ComputeAlignedRange()
raw_size = raw_range.GetMax() - raw_range.GetMin()
real_size_meters = [raw_size[i] * mpu for i in range(3)]
```

## Transform matrix order

When placing an asset with scale + rotation + translation, author the matrix on a new wrapper. Keep the referenced asset beneath it so its own transforms survive:
```python
from pxr import Gf, UsdGeom

xf = UsdGeom.Xform.Define(stage, "/World/Placement")  # New path owned by this placement
model = stage.DefinePrim("/World/Placement/Asset", "Xform")
model.GetReferences().AddReference(asset_path)
# Gf uses row vectors: point * scale * rotation * translation.
ratio = asset_mpu / stage_mpu
scale_mat = Gf.Matrix4d().SetScale(Gf.Vec3d(ratio, ratio, ratio))
rot_mat = Gf.Matrix4d().SetRotate(Gf.Rotation(Gf.Vec3d(0, 0, 1), heading_deg))
trans_mat = Gf.Matrix4d().SetTranslate(Gf.Vec3d(x, y, z))
xf.MakeMatrixXform().Set(scale_mat * rot_mat * trans_mat)
```

`translate * scale` scales the translation too. For example, a 3-unit offset becomes 0.03 with a 0.01 scale. This order follows the [Gf matrix convention](https://openusd.org/release/api/class_gf_matrix4d.html); do not copy column-vector formulas without transposing their order.

## Placement Helper

```python
def place(stage, prim_path, asset_path, x, y, z=0, rot_z=0):
    """Place a Z-up asset under a new wrapper; keep its authored transforms."""
    from pxr import Gf, Usd, UsdGeom

    asset_stage = Usd.Stage.Open(asset_path)
    if not asset_stage or not asset_stage.GetDefaultPrim():
        raise ValueError("Asset needs a readable stage and default prim")
    if UsdGeom.GetStageUpAxis(asset_stage) != UsdGeom.GetStageUpAxis(stage):
        raise ValueError("Convert the asset up-axis before using this helper")
    if UsdGeom.GetStageUpAxis(stage) != UsdGeom.Tokens.z:
        raise ValueError("This helper defines heading around Z")
    if stage.GetPrimAtPath(prim_path):
        raise ValueError("Choose a new placement path")
    ratio = UsdGeom.GetStageMetersPerUnit(asset_stage) / UsdGeom.GetStageMetersPerUnit(stage)
    wrapper = UsdGeom.Xform.Define(stage, prim_path)
    model = stage.DefinePrim(prim_path + "/Asset", "Xform")
    model.GetReferences().AddReference(asset_path)
    scale = Gf.Matrix4d().SetScale(Gf.Vec3d(ratio))
    rotation = Gf.Matrix4d().SetRotate(Gf.Rotation(Gf.Vec3d(0, 0, 1), rot_z))
    translation = Gf.Matrix4d().SetTranslate(Gf.Vec3d(x, y, z))
    wrapper.MakeMatrixXform().Set(scale * rotation * translation)
    return wrapper.GetPrim()
```

This places the asset origin in the wrapper's parent space. For an offset origin, ground contact, or a transformed parent, use [bounds and support-point placement](spatial-reasoning-advanced.md#bbox-offset-correction).

## Look-At Camera Math

```python
def look_at_rotation(cam_pos, target_pos):
    """Compute XYZ Euler rotation for camera to look at target. Z-up stage."""
    import math
    from pxr import Gf

    dx = target_pos[0] - cam_pos[0]
    dy = target_pos[1] - cam_pos[1]
    dz = target_pos[2] - cam_pos[2]
    if dx == dy == dz == 0:
        raise ValueError("Camera and target must differ")
    horiz = math.sqrt(dx * dx + dy * dy)
    pitch = math.degrees(math.atan2(horiz, -dz))
    yaw = math.degrees(math.atan2(-dx, dy))
    return Gf.Vec3f(pitch, 0, yaw)
```

## Grid Layout

```python
def grid_positions(n_items, spacing, origin=(0, 0)):
    """Generate grid positions for n items with given spacing."""
    import math

    if n_items <= 0:
        return [], 0
    cols = min(8, math.ceil(math.sqrt(n_items * 1.5)))
    positions = []
    for i in range(n_items):
        row, col = divmod(i, cols)
        x = origin[0] + col * spacing
        y = origin[1] + row * spacing
        positions.append((x, y))
    return positions, cols
```

## Baked waypoint animation

_See `bake_waypoints()` in [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py) (28 lines)._

This keyframes an XformOp. It is visualization/replay, not physical locomotion or manipulation. Do not convert a physics task into baked animation.


## Zone Boundary Check

```python
def in_zone(x, y, zone):
    """Check if (x,y) is within a zone dict with 'x':[min,max], 'y':[min,max]."""
    return zone["x"][0] <= x <= zone["x"][1] and zone["y"][0] <= y <= zone["y"][1]
```

## Shell vs interior camera alignment

Warehouse shells are often much larger than the interior layout you actually placed. When placing interior assets at zone coordinates, put the camera **inside the layout zone**, not at the building origin (which may sit in wall geometry).

Example starting points (retune from measured bounds):
- **Hero camera:** `(zone_start_x + 3, zone_center_y, 2.5)` looking into the zone.
- **Top-down:** `(layout_center_x, layout_center_y, max(W, D) * 0.9)` with a short, wide-angle focal length (example 12 mm).

Before capturing, verify that the camera sees the placed content and does not intersect walls or equipment. It may be outside the content bounds for an overview. Viewport look-at helpers do not replace sensor cameras ([sensors](isaac-sim-sensor.md), [isaac-camera.md](isaac-camera.md)). USD cameras look along −Z; Isaac `camera_axes="world"` is +X forward / +Z up.

## Render diagnosis (not file-size laws)

PNG size, hashes, mean, and variance are not quality oracles. A dark or white scene can be intentional. If intended views produce identical images, inspect the active camera, pose, and frame timestamp. Rebind the capture source or recreate an owned camera when it is stale, then inspect a new frame. A fixed settle-frame count does not prove renderer readiness.

## Mixed-unit asset catalogs

Catalog trees often mix meters and centimeters. Always validate with `Usd.Stage.Open()` + `UsdGeom.BBoxCache`; do not assume a vendor folder uses one unit.

```python
stage = Usd.Stage.Open(asset_path)
bbox = UsdGeom.BBoxCache(0, [UsdGeom.Tokens.default_]).ComputeWorldBound(stage.GetPseudoRoot()).ComputeAlignedRange()
size = bbox.GetMax() - bbox.GetMin()
# Heuristic only: a dimension > 1000 in a supposed-meter asset often means centimeters (mpu=0.01).
# Prefer the authored metersPerUnit over this size heuristic.
```

Placing meter-scale assets with an extra 0.01 scale makes them 100× too small (invisible). Assets "exist" in USD but render as sub-centimeter specks.

## Common Gotchas

- A shell's outer bounds do not describe its walkable interior. Inspect the floor, walls, openings, and module children.
- An unmodified `UsdGeom.Cube` has size 2. To make dimensions `(w, d, h)`, use scales `(w/2, d/2, h/2)` or set its size explicitly.
- `Gf.Matrix4d.SetTranslateOnly()` preserves the non-translation entries. `SetTranslate()` replaces the matrix with a translation. Keep rotation and scale when moving existing geometry.
- A world-space displacement differs from a child-local displacement under a rotated or scaled parent. Convert through the parent's inverse before changing local translation.
- Read the composed stage for bounds. Binary crate layers are supported by Sdf.
- With Gf row vectors, apply scale, rotation, and translation as `S * R * T`.
- USD cameras look along local -Z. On a Z-up stage, an unrotated camera points down.
- `xformOp:translate already exists in xformOpOrder` means a translate op is already authored. Reuse that op or place a new wrapper above the asset. Clearing an imported transform stack can destroy asset scale and orientation.
- Rebuild or clear transform and bound caches after edits. During simulation use live physics state for physical measurements.
- Renderer warm-up depends on assets, shaders, and temporal accumulation. Check fresh frame content within a deadline.

### Layout scaling
- When placing large equipment, check `equipment_area / facility_area`. Upstream used 30% as a review trigger for one large module, not a universal maximum. Check actual clearances; if a module cannot fit, grow the facility or split the module instead of overlapping zones.
- Before placing a new zone, check existing zone bounding boxes. Overlap of AABB ranges is a layout clash, not PhysX contact.
- An occupancy grid after a layout pass is a spatial sanity check, not a navigation-success metric. There is no universal free-space percentage.

## Compare layout variations

Develop one variation, render the important views, inspect them, then adjust the layout. Give each variation a clear purpose, such as shorter travel or more staging space. Compare from the same cameras and rerun bounds and occupancy checks after each change. For larger layouts, continue with the advanced reference linked above.
