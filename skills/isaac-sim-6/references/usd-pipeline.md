# USD Asset Pipeline

Adapted from NVIDIA [`usd-pipeline/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-pipeline/SKILL.md) (Apache-2.0).

[Isaac Sim task index](../SKILL.md#domain-references) · [Antioch startup](../../antioch-platform/references/simulation-code.md)

See also [usd-composition-architecture.md](usd-composition-architecture.md) and [spatial-reasoning.md](spatial-reasoning.md).

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

Use this progression when replacing a rough layout with reusable USD assets:

1. **Prototype with cubes** — Layout spatial zones using colored UsdGeom.Cube meshes
2. **Catalog assets** — Measure every candidate USD asset (bbox, shaders, prim count)
3. **Map blocks to assets** — Match placeholder types to appropriately-sized real assets
4. **Place with offset correction** — Reference assets at block positions, correcting for asset bbox center offset
5. **Validate the scene** — Check bounds, clearances, materials, and fresh renders against the task
6. **Iterate** — Fix overshoot, corridor intrusion, scale mismatches

### Why Bbox Offset Correction Matters

An asset origin may be far from its geometry. A rack asset might have its bbox center at `(46.7, 104.6, 4.5)` — if you place it at the target position `(112.0, 20.0, 0.0)` without correction, it lands 46.7m east and 104.6m north of where you want it.

**Unrotated formula:**
```
translate_x = target_x - asset_bbox_center_x
translate_y = target_y - asset_bbox_center_y
translate_z = support_z - asset_bbox_min_z
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

- **Use `Usd.Stage.Open()` for composed measurements.** Sdf supports binary crate layers, but a layer alone does not compose referenced geometry.
- **Always check mpu** — Some assets use cm (mpu=0.01), some use meters (mpu=1.0). Scale measurements accordingly.
- **Empty or nonfinite bounds** need investigation. Check missing references, unloaded payloads, selected purposes, and whether the prim has bounded geometry.
- **Offline `Usd.Stage.Open`** is enough for many measurements; a Kit process is not required just to read metersPerUnit.
- **For bounds in a live stage**, measure a temporary reference you own, then remove it. Rebuild caches after edits and validate the composed bounds. Do not advance an unrelated simulation just to inspect an asset.

## Phase 2: Shader Compatibility Check

### Renderer compatibility

RTX supports MDL in headless sessions. A missing UsdPreviewSurface fallback does not make an asset unusable. Check what the target renderer supports and whether material dependencies resolve there:

| Material | Inspect |
|---|---|
| UsdPreviewSurface | Shader inputs, textures, UVs, and material bindings |
| MDL | MDL module resolution, subidentifier, textures, bindings, and shader compilation logs |
| Multiple render contexts | Which shader output the renderer selects; a fallback's presence does not prove it is bound |
| No material | Whether a plain surface is intended or a binding is missing |

Upstream reported black MDL-only assets in its headless ARM64 setup. Keep that symptom in the diagnosis, but do not generalize it into a headless MDL ban: inspect module resolution, compilation logs, bindings, and the target renderer.

A fallback can improve portability across renderers. `info:id = "UsdPreviewSurface"` identifies a shader, and `info:mdl:sourceAsset` identifies an MDL source asset; neither alone proves that the rendered object uses it. Inspect a fresh image with the intended camera and lighting. Flattening composition does not embed external textures.

Use [Antioch startup](../../antioch-platform/references/simulation-code.md), then the [rendering guide](isaac-sim-rendering.md) for capture and lighting. Do not start a second `SimulationApp` or a local simulator.

## Phase 3: Placeholder-to-Asset Mapping

### Strategy

1. Group placeholder cubes by name prefix (e.g., `CvL001`→`CvL`, `BRk045`→`BRk`)
2. For each prefix, find the best-fit asset by:
   - Similar function (racks→rack assets, conveyors→conveyor assets)
   - Compatible size (asset shouldn't massively overshoot the placeholder zone)
   - Materials supported by the target renderer with resolvable dependencies
3. Document the mapping table before building

### Mapping Table Format

```
| Block Prefix | Count | Placeholder Size | Asset | Asset Size | Shader | Notes |
|---|---|---|---|---|---|---|
| BRk | 352 | 3.5×1.2×5.0m | ASRS_Racks_Center | 2.36×2.82×8.29m | dual | Per-block, no scaling |
| CvL | 190 | 2.0×4.0×1.5m | Conveyor_09 | 0.94×6.72×1.66m | dual | Roller conveyor |
```

### Size Philosophy

Prefer assets whose natural dimensions fit the task. Scaling can be valid when the model calls for it, but changes collision geometry and may require new mass and inertia. Do not stretch a robot or mechanism merely to fill a proxy.

If an asset is dramatically larger than its zone (e.g., 83m assembly in a 20m zone), use smaller modular pieces instead.

## Phase 4: Placement Script Pattern

### Runtime placement

`get_asset_bbox(stage, asset_path, app)` — reference asset temporarily to get accurate bbox center. `collect_blocks(stage)` — group visible Cube prims by name prefix with world positions. `place_assets(stage, blocks, asset_map, asset_bboxes, module_name)` — place assets at block positions with bbox-center offset correction; hides original cubes.

See [`scripts/place_assets.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-pipeline/scripts/place_assets.py).

The upstream `get_asset_bbox(..., app)` helper calls `app.update()`. On a playing timeline this can advance the simulation. For measurement alone, open a separate composed USD stage or adapt the helper to avoid stepping an unrelated running scene.

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
- Preserve valid authored UVs. Choose a mapping that fits the geometry; check seams, tiling, scale, and orientation.
- Provide a `UsdPreviewSurface` fallback when the asset must work in renderers that need it
- Verify the actual material in the target renderer, including headless RTX

## Phase 6: Validation

### Color Key for Placeholder Flow
| Color | Value | Represents |
|-------|-------|------------|
| Red | (1.0, 0.2, 0.2) | Failed validation
| Yellow | (1.0, 1.0, 0.2) | Warning (near overlap)
| Green | (0.2, 1.0, 0.2) | Valid, ready to replace with asset

### Packaging and validation

- Check the default prim, units, up-axis, references, payloads, variants, textures, and material bindings.
- Preserve valid relative paths such as `../materials/paint.usda` within the delivered package. Do not prohibit relative references or assume flattening embeds their dependencies.
- Validate transformed bounds and clearances after replacing proxies. Keep visual inspection separate from physics contact checks.
- Reopen the package from the destination layout so references cannot succeed only through files left in the source checkout.
- Use [asset upload and download](../../antioch-platform/references/assets.md) to publish and verify the complete asset package.

### Choosing candidates

Catalog the library before choosing replacements. Match function, dimensions, collision detail, articulation requirements, materials, and rendering cost. A prim-count threshold or fixed size tolerance cannot decide every task. Inspect whether a catalog entry is a finished asset, a proxy, a single module, or an entire assembly.

## Common failures

1. An origin offset places otherwise correctly sized geometry outside its zone. Check the transformed bounds, not just the prim position.
2. A whole assembly replaces one module. Inspect child prims or choose a smaller reference.
3. A second unit conversion makes an asset 100 times too small. Record stage and source units and apply their ratio once.
4. A black render can come from framing, missing lights, unresolved material dependencies, or an unready capture. Inspect the decoded frame and logs.
5. Replacing a transform stack removes authored scale or orientation. Put layout placement on a wrapper when possible.
6. Baked animation is useful for replay, but does not demonstrate physical navigation or manipulation. For physical robots, continue with [articulations](usd-articulation.md), [navigation](navigation-primitives.md), or [manipulation](manipulation-ik.md).
7. Viewport capture, a sensor camera, and a Replicator render product have different purposes. See [rendering](isaac-sim-rendering.md) and [sensors](isaac-sim-sensor.md).
