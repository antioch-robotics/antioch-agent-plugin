# USD Asset Pipeline

Adapted from NVIDIA [`usd-pipeline/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-pipeline/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Discover assets, measure bounds and shaders, swap placeholders, correct offsets, and validate shader compatibility before rendering.

See also [usd.md](usd-composition-architecture.md), [usd-composition-architecture.md](usd-composition-architecture.md), [spatial-reasoning.md](spatial-reasoning.md).

## Upstream helpers (source examples)

| Script | Purpose |
|---|---|
| [`scripts/articulation_builder.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-pipeline/scripts/articulation_builder.py) | Helpers for building USD articulation trees with physics joints |
| [`scripts/measure_asset.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-pipeline/scripts/measure_asset.py) | Measure a USD asset: bounding box, shader type, and prim count |
| [`scripts/place_assets.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-pipeline/scripts/place_assets.py) | Place USD assets at block positions with bbox-center offset correction |

Not bundled commands.

## When to use

- Catalog USD assets from a folder tree (sizes, shaders, prim counts).
- Replace placeholder geometry (cubes/spheres/bboxes) with real assets.
- Build set-dressed scenes from modular libraries.
- Validate headless rendering compatibility (MDL vs UsdPreviewSurface).
- Cube-prototype to real-asset swap workflows.

## Core Concepts

### The Placeholder-to-Asset Pipeline

Real-world USD scene building follows this pattern:

1. **Prototype with cubes** — Layout spatial zones using colored UsdGeom.Cube meshes
2. **Catalog assets** — Measure every candidate USD asset (bbox, shaders, prim count)
3. **Map blocks to assets** — Match placeholder types to appropriately-sized real assets
4. **Place with offset correction** — Reference assets at block positions, correcting for asset bbox center offset
5. **Validate renders** — Vision model + domain expert scoring
6. **Iterate** — Fix overshoot, corridor intrusion, scale mismatches

### Why Bbox Offset Correction Matters

Most USD assets are NOT centered at origin. A rack asset might have its bbox center at `(46.7, 104.6, 4.5)` — if you place it at the target position `(112.0, 20.0, 0.0)` without correction, it lands 46.7m east and 104.6m north of where you want it.

**Unrotated formula:**
```
translate_x = target_x - asset_bbox_center_x
translate_y = target_y - asset_bbox_center_y
translate_z = -asset_bbox_min_z  (puts asset base on ground plane)
```

After rotation or nonuniform scale, transform all eight bbox corners rather than subtracting an unrotated center. USD references do not convert units or axes. For an unscaled asset:

```text
scale = asset_meters_per_unit / stage_meters_per_unit
```

Apply conversion once, including any importer or parent scale. Use collision geometry for clearance. `UsdGeom.BBoxCache` and `UsdGeom.XformCache` inspect composed USD; clear or rebuild them after edits and do not treat them as live physics state.

## Phase 1: Asset Discovery & Measurement

### Script Pattern (Kit Python — No Renderer Needed)

`measure_asset(path)` — open a USD file, compute bbox, detect UsdPreviewSurface shader, count prims.  Returns dict with width/depth/height/center/mpu/prims/dual_shader/shader_tag.  `catalog_assets(root_dir, extensions)` — recursively find and measure all USD assets.

See [`scripts/measure_asset.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-pipeline/scripts/measure_asset.py).

### Key Learnings

- **Use `Usd.Stage.Open()` not `Sdf.Layer.FindOrOpen()`** — Sdf fails silently on binary .usd crate files, returns default mpu=1.0
- **Always check mpu** — Some assets use cm (mpu=0.01), some use meters (mpu=1.0). Scale measurements accordingly.
- **Invalid bbox (min > 1e30)** means the asset didn't compose — usually missing references or payloads
- **Offline `Usd.Stage.Open`** is enough for many measurements; a Kit process is not required just to read metersPerUnit.
- **For bbox of composed references in a live stage**, define a temp prim with a reference, step the app a few times so composition resolves, then compute bbox. A fixed frame count does not prove readiness — check that the bound is finite and not the empty/infinity sentinel (`min > 1e30`).

## Phase 2: Shader Compatibility Check

### The MDL problem without a GUI viewport

| Shader Type | Remote/headless RTX | Isaac Sim GUI | OVRTX |
|---|---|---|---|
| UsdPreviewSurface only | renders | renders | renders |
| MDL + UsdPreviewSurface (dual) | may fall back to Preview | uses MDL | uses MDL |
| MDL only (sourceAsset) | can be **black** | renders | renders |
| No materials | grey/invisible | grey | grey |

**Rule:** For pipelines without GUI look-dev, prefer assets with UsdPreviewSurface fallback (dual-shader) or native UsdPreviewSurface. RTX supports MDL, but material bindings, asset resolution, shader logs, lights, camera, and renderer still need a real frame check. Flattening composition does not embed external textures.

Do not start a local Xvfb, `isaac-sim.sh`, or `SimulationApp`. Antioch runs the remote simulator; add explicit lights when capturing (GUI viewport lights are not present).

### Identifying dual-shader assets

Look for these patterns in USD:
- `info:id = "UsdPreviewSurface"` on any Shader prim → headless-safe
- `info:mdl:sourceAsset` without UsdPreviewSurface sibling → MDL-only, headless-unsafe
- Some processed catalog assets are MDL-only; collected dual-shader variants are safer for non-GUI capture. Inspect the actual USD rather than trusting a vendor nickname.

## Phase 3: Placeholder-to-Asset Mapping

### Strategy

1. Group placeholder cubes by name prefix (e.g., `CvL001`→`CvL`, `BRk045`→`BRk`)
2. For each prefix, find the best-fit asset by:
   - Similar function (racks→rack assets, conveyors→conveyor assets)
   - Compatible size (asset shouldn't massively overshoot the placeholder zone)
   - Dual-shader compatibility (headless rendering requirement)
3. Document the mapping table before building

### Mapping Table Format

```
| Block Prefix | Count | Placeholder Size | Asset | Asset Size | Shader | Notes |
|---|---|---|---|---|---|---|
| BRk | 352 | 3.5×1.2×5.0m | ASRS_Racks_Center | 2.36×2.82×8.29m | dual | Per-block, no scaling |
| CvL | 190 | 2.0×4.0×1.5m | Conveyor_09 | 0.94×6.72×1.66m | dual | Roller conveyor |
```

### Size Philosophy

**Use natural asset sizes, NOT scaled-to-cube.** Scaling assets to match cube dimensions destroys visual density and realism. Place at the block's XY position with the asset's natural dimensions.

Exception: If an asset is dramatically larger than its zone (e.g., 83m assembly in a 20m zone), use smaller modular pieces instead.

## Phase 4: Placement Script Pattern

### Runtime placement

`get_asset_bbox(stage, asset_path, app)` — reference asset temporarily to get accurate bbox center. `collect_blocks(stage)` — group visible Cube prims by name prefix with world positions. `place_assets(stage, blocks, asset_map, asset_bboxes, module_name)` — place assets at block positions with bbox-center offset correction; hides original cubes.

See [`scripts/place_assets.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-pipeline/scripts/place_assets.py).

### Hierarchy Convention

```
/World/
  Module1/          # Racks
    VNA/
      VNA001        # Individual asset reference
      VNA002
    BRk/
      BRk001
  Module2/          # Conveyors, sorters
    CvL/
      CvL001
    Sort/
      Sort001
  Module3/          # Safety, humans
```

## Phase 5: Animation & Articulation

### Animation Baking

Use `bake_waypoints()` from [spatial-reasoning.md](spatial-reasoning.md) to **keyframe** robots, humans, or objects along paths. That is visualization/replay, not physical locomotion. Do not convert a physics navigation or manipulation task into baked animation and call it success.

`bake_waypoints(xform_op, waypoints, speed_mps, fps, mpu)` — keyframe an XformOp along a list of (x,y,z) waypoints. Source: [`scripts/spatial.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/spatial-reasoning/scripts/spatial.py).

### Articulation Builder

For robot USD:

`build_forklift_articulation(output_path)` — create chassis + fixed-joint mast + prismatic lift joint; includes `_create_box`, `_create_fixed_joint`, `_create_prismatic_joint` helpers.

See [`scripts/articulation_builder.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-pipeline/scripts/articulation_builder.py).

### UV Mapping & Materials

For textures:
- Use `sphereUV` mapping for global assets
- Use `linear` for flat planes (floors, walls)
- Always use `UsdPreviewSurface` as fallback
- Never use MDL-only materials for headless

## Phase 6: Validation

### Color Key for Placeholder Flow
| Color | Value | Represents |
|-------|-------|------------|
| Red | (1.0, 0.2, 0.2) | Failed validation
| Yellow | (1.0, 1.0, 0.2) | Warning (near overlap)
| Green | (0.2, 1.0, 0.2) | Valid, ready to replace with asset

### Format Validation Checklist
- [ ] All assets have `.usd`, `.usda`, or `.usdc` extension
- [ ] All paths use forward slashes `/`
- [ ] No `..` relative paths
- [ ] Stage and asset `metersPerUnit` recorded; scale applied once (`asset_mpu / stage_mpu`)
- [ ] No `./` or `../` syntax in references
- [ ] Topology: artifacts must not be nested under `Xform` if empty

### Asset Recommendation (Based on Scan)

First, scan all assets with `catalog_assets()`, then:

- Filter: `dual_shader` is True
- Sort: `prims < 5000`
- Assign: Match bbox dimensions within 20% tolerance of placeholder cube
- Reject: catalog entries that are placeholder stacks rather than real assets (inspect the USD)

## Hard-Won Lessons

1. **Never scale assets to match cube dimensions** — destroys visual density. Use natural sizes.
2. **Large assemblies (>20m) rarely fit block clusters** — use smaller modular pieces instead.
3. **Always correct for bbox center offset** — most assets aren't origin-centered.
4. **MDL-only materials can render black** without GUI look-dev. Prefer dual-shader / UsdPreviewSurface fallback. Named catalog assets (totes, pallet piles, tables) must be inspected, not assumed.
5. **Do not kill Kit processes, clean `/dev/shm/carb-*`, or relaunch locally.** Antioch owns the remote simulator process.
6. **Remote/headless capture requires explicit DomeLight + DistantLight** — a GUI viewport adds lights automatically; the remote session does not.
7. Viewport capture (`antioch.capture_viewport` / `capture_viewport_to_file`) is not a sensor camera or Replicator product. See [rendering.md](isaac-sim-rendering.md) / [sensors.md](isaac-sim-sensor.md).
8. File-size heuristics (KB/MB) are not quality oracles. Inspect the decoded frame.
