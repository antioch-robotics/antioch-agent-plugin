# Measuring & placing USD assets

Depth reference for the assets domain of the `isaac-sim-6` skill. All of this
runs on the remote machine. Measurement needs no renderer or simulator boot
(pure `pxr` stage reads); placement inside a live scene happens in scenario
code after boot.

## metersPerUnit — read it before trusting any number

Isaac Sim stages are meters, Z-up. Third-party assets are often authored in
centimeters. `metersPerUnit` (mpu) is a stage metadata value, not a hint:

```python
def read_mpu(usd_path: str) -> float:
    from pxr import Usd, UsdGeom

    # Open the composed stage, not a raw layer: Sdf.Layer.FindOrOpen skips
    # composition, so references/payloads never resolve — and
    # GetStageMetersPerUnit takes a stage anyway.
    stage = Usd.Stage.Open(usd_path)
    return UsdGeom.GetStageMetersPerUnit(stage)
```

- Common values: `1.0` (meters — robots, most assemblies), `0.01`
  (centimeters — large shells, lighting rigs, some vendor libraries).
- Heuristic when the metadata lies: if any raw bbox dimension exceeds
  ~1000 units, the asset is almost certainly authored in cm.
- Real-world size is `raw_size * mpu`. Placing a cm asset without scaling
  makes it 100× too small — it "exists" in USD but renders as a
  sub-centimeter speck.
- Never edit mesh data to fix units; wrap the asset in a parent xform
  scaled by mpu.

## Full measurement function

Extends the `measure_asset` sketch in the SKILL with prim count and the
headless shader check:

```python
def measure_asset(usd_path: str) -> dict:
    from pxr import Usd, UsdGeom, UsdShade

    stage = Usd.Stage.Open(usd_path)
    prim = stage.GetDefaultPrim() or stage.GetPseudoRoot()
    mpu = UsdGeom.GetStageMetersPerUnit(stage)
    bbox = UsdGeom.BBoxCache(Usd.TimeCode.Default(), [UsdGeom.Tokens.default_])
    extent = bbox.ComputeWorldBound(prim).ComputeAlignedRange()
    lo, hi = extent.GetMin(), extent.GetMax()
    if lo[0] > 1e30:
        raise ValueError(f"Asset '{usd_path}' did not compose — check its references and payloads")

    has_preview_surface = False
    has_mdl = False
    for p in stage.Traverse():
        if not p.IsA(UsdShade.Shader):
            continue
        info_id = p.GetAttribute("info:id")
        if info_id and info_id.Get() and "Preview" in str(info_id.Get()):
            has_preview_surface = True
        if p.GetAttribute("info:mdl:sourceAsset"):
            has_mdl = True

    return {
        "size_m": tuple((hi[i] - lo[i]) * mpu for i in range(3)),
        "center": tuple((lo[i] + hi[i]) / 2 for i in range(3)),
        "base_z": lo[2],
        "mpu": mpu,
        "prims": sum(1 for _ in stage.Traverse()),
        "headless_safe": has_preview_surface,
        "shader_tag": "dual" if has_preview_surface and has_mdl else ("preview" if has_preview_surface else "mdl-only"),
    }
```

Batch over a library:

```python
def catalog_assets(usd_paths: list) -> dict:
    return {p: measure_asset(p) for p in usd_paths}
```

For complex compositions that only resolve once Kit is live, measure inside
a running scenario instead: reference the asset on a temporary prim, step a
few frames, compute the bbox, remove the prim.

## Transform order — scale, rotate, translate

```python
def place(stage: "Usd.Stage", prim_path: str, asset_path: str, x: float, y: float, z: float, rot_z: float, mpu: float) -> None:
    from pxr import Gf, UsdGeom

    prim = stage.DefinePrim(prim_path, "Xform")
    prim.GetReferences().AddReference(asset_path)
    scale = Gf.Matrix4d().SetScale(Gf.Vec3d(mpu, mpu, mpu))
    rotate = Gf.Matrix4d().SetRotate(Gf.Rotation(Gf.Vec3d(0, 0, 1), rot_z))
    translate = Gf.Matrix4d().SetTranslate(Gf.Vec3d(x, y, z))
    UsdGeom.Xformable(prim).MakeMatrixXform().Set(scale * rotate * translate)
```

Gf uses row vectors: `A * B` applies `A` first, so the product is
`scale * rotate * translate`. `translate * rotate * scale` is wrong — the
trailing scale multiplies the translation vector, so a 3 m offset becomes
3 cm at mpu=0.01. `MakeMatrixXform()` clears any existing
xform ops — use it rather than `AddTranslateOp()` when the asset may
already carry ops (`xformOpOrder` conflicts) or when you need scale and
translate to coexist.

## Offset-corrected placement

Most assets are not origin-centered. Placing at the target without
correction lands the asset wherever its bbox center happens to be. Combine
with the measured bbox:

```python
def place_corrected(stage: "Usd.Stage", prim_path: str, asset_path: str, target_xy: tuple) -> None:
    info = measure_asset(asset_path)
    tx = target_xy[0] - info["center"][0]
    ty = target_xy[1] - info["center"][1]
    tz = -info["base_z"]  # grounds the asset on z=0
    place(stage, prim_path, asset_path, tx, ty, tz, rot_z=0.0, mpu=info["mpu"])
```

Use natural asset sizes — do not scale an asset to match a placeholder
cube's dimensions; it destroys visual density and collision fidelity. If an
asset is dramatically larger than its zone, use smaller modular pieces.

## Remote-rendering shader audit

Antioch renders on the remote machine, whether or not its GUI is streaming to
the browser. MDL-only materials render black in that server-side renderer; the
fix is the asset, not a local display or framebuffer setting.

| Shader authored | Headless result |
|---|---|
| `UsdPreviewSurface` | renders |
| MDL + `UsdPreviewSurface` fallback ("dual") | renders (fallback) |
| MDL only (`info:mdl:sourceAsset`, no Preview sibling) | black |
| No materials | grey |

Detection patterns:

- `info:id = "UsdPreviewSurface"` on any Shader prim → headless-safe.
- `info:mdl:sourceAsset` present without a `UsdPreviewSurface` sibling →
  MDL-only, headless-unsafe.
- Vendor "processed"/"baked" asset variants are often MDL-only; "collected"
  variants more often carry the dual shader. Measure, don't assume.

Filter a catalog before committing to an asset set: keep
`headless_safe=True`, prefer `prims < 5000` for scene dressing, and match
bbox dimensions to the placeholder zone within ~20%.
