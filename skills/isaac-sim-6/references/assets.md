# Import, inspect, and place assets

Use this for URDF/MJCF conversion, USD robot preparation, and measured
placement. The parent skill owns the runtime pin and lazy imports.

## Choose the input

Respect the asset the user chose. For a named catalog asset, use the platform
skill's asset workflow and pin the resolved version. For an existing project
file, inspect that file. A primitive is a useful diagnostic fixture, but not a
substitute for a requested robot or environment.

Keep dependencies with the asset. A USD that references meshes, textures,
payloads, or material files is not portable as a lone file. Opening it without
an exception does not prove that those dependencies composed successfully.

The stock Franka path used by the pinned examples is
`/Isaac/Robots/FrankaRobotics/FrankaPanda/franka.usd`, relative to
`isaacsim.storage.native.get_assets_root_path()`. Check that the root and
asset resolve in the selected image; do not assume a private catalog asset
exists in every organization.

## Conversion

The pinned import stack is `isaacsim.asset.importer.urdf` and
`isaacsim.asset.importer.mjcf`. Research the chosen importer's config and
command signature before use; the two importers do not share every option.
Isaac Lab also provides `isaaclab.sim.converters` wrappers.

1. Resolve referenced mesh and package paths. Expand XACRO before URDF import.
2. Choose fixed or floating base to match the task. Inspect defaults rather
   than treating a tri-state option as a simple boolean.
3. Set collision approximation, drive mode, limits, density/mass, and
   instanceability deliberately. Importer defaults are not robot calibration.
4. Write to a new, owned output directory. Do not erase the source or an
   existing conversion to clear a cache.
5. Inspect the composed USD and run a small physical probe before building a
   controller around it.

## Composition and units

Read the asset's `metersPerUnit`, up-axis, default prim, transform stack,
and composed bounds. USD references do not automatically convert units or
up-axis.

For an unscaled asset whose coordinates are in asset units, the uniform
conversion into the destination stage is:

```text
scale = asset_meters_per_unit / stage_meters_per_unit
```

A centimeter-authored object referenced into a meter stage without that
conversion is 100 times too large, not too small. Apply conversion once;
account for any scale already authored by an importer or parent transform.

Measure bounds in a known frame before placing the asset. For zero rotation,
multiply both dimensions and center/base offsets by the conversion scale
before computing the placement translation. With rotation, transform the
corners and recompute bounds; subtracting an unrotated offset is wrong.
Use actual collision geometry when reasoning about physical clearance.

`UsdGeom.BBoxCache` and `UsdGeom.XformCache` can inspect composed USD state.
Clear/rebuild caches after edits. They do not independently read a physics
backend's live tensors; the parent skill explains runtime readback.

## Robot checks

Validate each intended articulation, not "exactly one root in the stage."
A scene may legitimately contain several robots.

- Confirm the selected articulation's root and links compose, including
  instance proxies when inspecting a referenced instance.
- Check joint body relationships, connectivity, frames, limits, names, and
  drive units against the actual robot.
- Do not require `ArticulationRootAPI` on a universal "chassis" path. Its valid
  location depends on fixed/floating-base structure and importer output.
  Retrieve the schema and the backend's articulation discovery rules.
- Look for unintended nested rigid bodies, missing colliders, and unwanted
  self-collision. Confirm collider approximation visually and with contacts.
- Measure mass/inertia and initial overlaps. An imported mesh that looks
  right can have a collision hull that fills a gripper opening.
- Instance proxies are read-only. Edit the source asset, use a deliberate
  override, or de-instance only the subtree that must differ. Do not
  de-instance an entire multi-environment scene by default.

After reset, verify finite live poses, expected DOF names/counts, gravity
response, a bounded joint command, and contact with the intended surface.
Record measurements and the asset/version used.

## Materials and delivery

RTX supports MDL; an MDL-only material is not inherently black or incompatible
with headless rendering. Inspect the selected renderer, material bindings,
asset resolution, shader errors, lights, and camera. A shader-name inventory
does not prove that a render works.

Keep a sample rendered frame and a physical check when both appearance and
collision matter. Flattening USD composition alone does not bundle external
textures. Validate the packaged asset in a fresh run before publishing it.
