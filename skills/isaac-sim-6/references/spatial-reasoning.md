# 3D Spatial Reasoning

Adapted from NVIDIA [`spatial-reasoning/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Apply coordinate math, bounding-box analysis, collision-free layouts, look-at placement, and lightweight path/grid helpers when composing USD scenes.

Geometric overlap of AABBs is not physical contact. A layout that looks clear in a top-down render is not a collision-free claim.

See also [usd.md](usd-composition-architecture.md), [usd-pipeline.md](usd-pipeline.md), [spatial-reasoning-advanced.md](spatial-reasoning-advanced.md).

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
asset_stage = Usd.Stage.Open(asset_path)  # Must use Usd.Stage, NOT Sdf.Layer
mpu = UsdGeom.GetStageMetersPerUnit(asset_stage)
```
`Sdf.Layer.FindOrOpen()` FAILS SILENTLY on binary .usd crate files — always returns None. Never use it for mpu detection.

**Getting real-world size of an asset:**
```python
bbox_cache = UsdGeom.BBoxCache(Usd.TimeCode.Default(), [UsdGeom.Tokens.default_])
raw_range = bbox_cache.ComputeWorldBound(default_prim).ComputeAlignedRange()
raw_size = raw_range.GetMax() - raw_range.GetMin()
real_size_meters = [raw_size[i] * mpu for i in range(3)]
```

## Transform matrix order

When placing an asset with scale + rotation + translation:
```python
# CORRECT: Translate * Rotate * Scale
# Scale shrinks asset to meters, Rotate orients it, Translate positions it
scale_mat = Gf.Matrix4d().SetScale(Gf.Vec3d(mpu, mpu, mpu))
rot_mat = Gf.Matrix4d().SetRotate(Gf.Rotation(Gf.Vec3d(0, 0, 1), heading_deg))
trans_mat = Gf.Matrix4d().SetTranslate(Gf.Vec3d(x, y, z))
xf.MakeMatrixXform().Set(trans_mat * rot_mat * scale_mat)
```

**WRONG:** `scale * translate` scales the translation vector too — a 3m offset becomes 0.03m for mpu=0.01.

## Placement Helper

```python
def place(stage, prim_path, asset_path, x, y, z=0, rot_z=0, mpu=0.01):
    """Place an asset with correct metersPerUnit scaling."""
    prim = stage.DefinePrim(prim_path, "Xform")
    prim.GetReferences().AddReference(asset_path)
    xf = UsdGeom.Xformable(prim)
    S = Gf.Matrix4d().SetScale(Gf.Vec3d(mpu, mpu, mpu))
    R = Gf.Matrix4d().SetRotate(Gf.Rotation(Gf.Vec3d(0, 0, 1), rot_z))
    T = Gf.Matrix4d().SetTranslate(Gf.Vec3d(x, y, z))
    xf.MakeMatrixXform().Set(T * R * S)
    return prim
```

## Look-At Camera Math

```python
def look_at_rotation(cam_pos, target_pos):
    """Compute XYZ Euler rotation for camera to look at target. Z-up stage."""
    dx = target_pos[0] - cam_pos[0]
    dy = target_pos[1] - cam_pos[1]
    dz = target_pos[2] - cam_pos[2]
    horiz = math.sqrt(dx * dx + dy * dy)
    pitch = math.degrees(math.atan2(horiz, -dz))
    yaw = math.degrees(math.atan2(dx, -dy))
    return Gf.Vec3f(pitch, 0, yaw)
```

## Grid Layout

```python
def grid_positions(n_items, spacing, origin=(0, 0)):
    """Generate grid positions for n items with given spacing."""
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
- **Top-down:** `(layout_center_x, layout_center_y, max(W, D) * 0.9)` with a wide focal length (example 12 mm).

Before capturing, verify camera pose is inside the bounding box of the placed content, not just inside the shell. Viewport look-at helpers do not replace sensor cameras ([sensors.md](isaac-sim-sensor.md), [isaac-camera.md](isaac-camera.md)). USD cameras look along −Z; Isaac `camera_axes="world"` is +X forward / +Z up.

## Render diagnosis (not file-size laws)

PNG size, hashes, mean, and variance are not quality oracles. A dark or white scene can be intentional. If several intended views produce identical bytes, the camera path likely did not switch — create a fresh camera prim for each shot (`stage.RemovePrim()` + `UsdGeom.Camera.Define()`), then inspect the decoded frame. A fixed settle-frame count does not prove renderer readiness.

## Mixed-unit asset catalogs

Catalog trees often mix meters and centimeters. Always validate with `Usd.Stage.Open()` + `UsdGeom.BBoxCache`; do not assume a vendor folder uses one unit.

```python
stage = Usd.Stage.Open(asset_path)
bbox = UsdGeom.BBoxCache(0, [UsdGeom.Tokens.default_]).ComputeWorldBound(stage.GetPseudoRoot()).ComputeAlignedRange()
size = bbox.GetMax() - bbox.GetMin()
# Heuristic only: a dimension > 1000 in a supposed-meter asset often means centimeters (mpu=0.01).
# Prefer the authored metersPerUnit over this size heuristic.
```

Placing meter-scale assets with an extra 0.01 scale makes them 100× too small (invisible). Assets "exist" in USD but render as sub-centimeter specs.

## Common Gotchas

0a. **Shell interior bounds ≠ shell bbox.** Always check module children bboxes, not just the shell root.
0b. **UsdGeom.Cube extent is [-1,1]³ (size 2)** — scale by `w/2, d/2, h/2` NOT `w, d, h`. Using full dimensions makes every block 2× its intended size.
0c. **`SetTranslateOnly()` wipes scale from Gf.Matrix4d** — if you set `mat[0][0]=0.01` then call `mat.SetTranslateOnly(...)`, the scale reverts to identity. Build the matrix with explicit 16 floats: `Gf.Matrix4d(sx,0,0,0, 0,sy,0,0, 0,0,sz,0, tx,ty,tz,1)`.
0d. **World delta != child local delta** — to move a child by a world vector while keeping its authored local rotation/scale, map the vector through the parent's inverse rotation/scale, then write only the translation row. Adding a world delta straight to the local translation applies the parent's rotation/scale to it (wrong motion whenever the parent isn't identity/translation-only).
1. `Sdf.Layer.FindOrOpen` fails on binary `.usd` crate files — use `Usd.Stage.Open`
2. Matrix order: `T * R * S` not `S * R * T`
3. Cameras default look along -Z in camera space = straight down in a Z-up stage (no rotation needed for top-down)
4. `xformOp:translate already exists in xformOpOrder` → use `MakeMatrixXform()` instead of `AddTranslateOp()`
5. Some character assets already have xformOps defined — `MakeMatrixXform()` clears and replaces
6. Layout area is usually a subset of the shell. Camera must be inside the layout area, not at origin.
7. Viewport `camera_path` can fail to switch between rapid captures — create a fresh camera prim per shot
8. Renderer warm-up depends on assets, shaders, and temporal accumulation. Use readiness/content checks and a bounded deadline, not a fixed frame count (for example "200 RT2 frames").

### Layout scaling
- When placing large equipment, check `equipment_area / facility_area`. If a module cannot fit, grow the facility or split the module — do not overlap zones.
- Before placing a new zone, check existing zone bounding boxes. Overlap of AABB ranges is a layout clash, not PhysX contact.
- An occupancy grid after a layout pass is a spatial sanity check, not a navigation-success metric. There is no universal free-space percentage.

---

## Advanced topics

See [spatial-reasoning-advanced.md](spatial-reasoning-advanced.md).
