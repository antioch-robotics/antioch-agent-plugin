# USD composition & stage authoring — Isaac Sim 6.0.1

Detail reference for the `isaac-sim-6` skill. All `pxr` imports belong inside
function bodies (the lazy-import invariant) — snippets here assume they run
inside a function with `stage` available.

## Stage basics

- Units: Isaac Sim stages are meters, Z-up (`metersPerUnit = 1.0`). Warehouse
  shells and third-party assets are often authored in cm — check
  `UsdGeom.GetStageMetersPerUnit(stage)` before placing, and wrap
  mismatched assets in a scale xform rather than editing mesh data.
- `/World` is the conventional root xform; put the `PhysicsScene` at
  `/World/PhysicsScene`.
- Always save the stage (`stage.GetRootLayer().Save()` or `Usd.Stage.Save`)
  when the artifact is the deliverable — never assume an in-memory stage
  survives.

## Adding assets

```python
ref = stage.OverridePrim("/World/Robots/Carter_0")
ref.GetReferences().AddReference(usd_path)
xf = UsdGeom.Xformable(ref)
xf.ClearXformOpOrder()
xf.AddTranslateOp().Set(Gf.Vec3d(x, y, z))
```

Grid placement: `row, col = divmod(i, cols)`, `x = col * spacing`,
`y = row * spacing`. Separation minimums: mobile robots 2 m between centers,
arms 1.5x reach radius, aerial stagger altitudes >= 2 m.

`stage.GetPrimAtPath(path)` returning an invalid prim means a case-sensitive
path typo or a missing reference — audit before assuming the asset broke.

## Composition arcs

USD resolves conflicting opinions by strength order — **LIVRPS**, strongest
first: **L**ocal (direct) opinions, **I**nherits, **V**ariantSets,
**R**eferences, **P**ayloads, **S**pecializes. Sublayers are not a separate
arc: they order the layer stack itself, with the root layer strongest.

- `reference` — graft an external asset; loads eagerly.
- `payload` — like a reference but lazy: only loaded on demand. This is the
  key to headless RL startup optimization (skip appearance payloads) — and the
  *weaker* arc, so a direct opinion beats it.
- `sublayer` — stack whole layers into the local layer stack; ordering
  between layers decides, not the arc list.
- `variantSet` — switchable configurations on a prim; stronger than the
  references/payloads it sits above.

If an edit "doesn't take effect", a stronger opinion is overriding it. Find
who sets a property with the property stack:

```python
attr = prim.GetAttribute("physics:mass")
for spec in attr.GetPropertyStack(Usd.TimeCode.Default()):
    print(f"  Layer: {spec.layer.GetDisplayName()} = {spec.default}")
```

## Layered asset structure (NVIDIA pattern)

One binary geometry crate plus hand-editable USDA layers, composed by an
`interface.usda` entry point:

```
{robot}/
  interface.usda      <- entry point: composition arcs
  payloads/
    base.usda         <- hierarchy + xforms
    geometries.usdc   <- mesh data ONLY (binary crate)
    instances.usda    <- mesh + material + collider assembly
    materials.usda    <- material defs (MDL bindings)
    Textures/
    robot.usda        <- Isaac robot schema metadata
    Physics/
      physics.usda    <- neutral USD physics (joints, masses)
      physx.usda      <- PhysX-only tuning, sublayers physics.usda
      mujoco.usda     <- MuJoCo-only tuning, sublayers physics.usda
```

Format rule: binary `.usdc` for raw mesh arrays (large, never hand-edited);
`.usda` for everything else (diffable, agent-editable); `.usdz` only for
single-file distribution.

| Layer | Format | Why |
|---|---|---|
| `geometries.usdc` | usdc | mesh topology/points — size + load speed |
| `base.usda` | usda | hierarchy, transforms — hand-editable |
| `instances.usda` | usda | references meshes, collision approximation |
| `materials.usda` | usda | MDL bindings, look-dev |
| `physics/physx/mujoco.usda` | usda | frequent tuning surface |
| `interface.usda` | usda | composition arcs, entry point |

## Headless / RL optimization

The biggest startup and VRAM win: don't load appearance payloads. Physics
only needs base + physics layers — skip `materials.usda` and `Textures/`.

- Mark repeated assets instanceable (shared mesh data across envs).
- Use `UsdGeom.PointInstancer` for large repeated sets (> 10K prims).
- Keep layer count low: thousands of per-asset layers dominate stage-open
  time; consolidate into a few library layers.

## Debugging checklist

| Symptom | Check |
|---|---|
| Edit ignored | property stack — higher-precedence layer overrides |
| Prim missing | case-sensitive path; missing/broken external reference |
| Wrong size | `metersPerUnit` mismatch (cm asset in m stage) |
| Slow stage open | layer count > 1000; consolidate into library layers |
| Broken refs hang load | audit `layer.GetExternalReferences()` for missing files |
