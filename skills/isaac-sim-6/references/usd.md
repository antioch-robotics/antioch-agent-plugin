# USD composition and stage authoring

Use this for layers, references, transforms, instancing, and missing or
overridden properties. Simulator imports remain inside functions.

## Inspect before editing

Read the stage's units, up-axis, default prim, layer stack, load rules, and
edit target. A missing prim can come from an inactive prim, unloaded payload,
variant selection, broken reference, or a wrong path.

`/World` and a meter/Z-up stage are common conventions, not universal USD
requirements. Referenced assets need explicit unit/up-axis conversion where
their conventions differ. The asset reference explains measured placement.

Create edits in the intended layer. Prefer a placement wrapper or a deliberate
override to clearing an asset's transform stack. `ClearXformOpOrder` can
discard an authored scale/orientation that makes the referenced asset correct.

## Composition

The usual LIVRPS strength mnemonic is local opinions, inherits, variants,
references, payloads, specializes. It helps orient an investigation; inspect
the actual property stack to resolve a concrete conflict.

References compose another asset. Payloads allow deferred loading, but can
also load eagerly under the stage's current load rules. Sublayers order the
local layer stack; they are not another slot in the mnemonic.

For an unexpected attribute value, inspect
`attribute.GetPropertyStack(Usd.TimeCode.Default())` and the active edit
target. Do not keep writing into a weaker layer and expect the result to win.

## Structure at real boundaries

Use separate geometry, material, physics, and composition layers when they
have separate owners or reuse needs. A small asset does not need a mandatory
directory tree. Binary USDC can suit large arrays; USDA is useful for reviewed
text edits. USDZ packaging has its own asset/localization constraints.

Instance repeated assets where sharing is valid. Instance proxies are
read-only; edit the source or de-instance only the part that needs unique
state. A point instancer is not interchangeable with separately simulated
articulations.

Skip appearance payloads only if the task does not need their geometry,
sensors, or rendered evidence. Do not remove visual data from a vision task
to make a memory check pass.

## Bounds, transforms, and runtime state

Choose local or world bounds deliberately. Clear or rebuild transform/bounds
caches after edits. Cache results can also become stale across simulation
steps. USD state is not an independent query into the live physics backend;
use the physics reference for authoritative runtime readback.

Validate transformed corners when placing rotated/scaled assets. Do not mix
local offsets, world positions, and stage units in one translation formula.

## Deliver the actual asset

Save to an owned output path when a stage is the deliverable. Anonymous
layers and in-memory edits do not survive process exit automatically.
Check which layers were saved; saving the root alone does not prove that all
referenced dependencies were packaged.

Open the delivered entry point in a fresh context and inspect dependencies,
bounds, materials, and any required physics. Flattening the stage does not
automatically embed external textures.
