# USD Composition Architecture for Isaac Sim

Adapted from NVIDIA [`usd-composition-architecture/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-composition-architecture/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Author sim-ready USD with layered payloads (base, instances, materials, physics, robot) following NVIDIA composition conventions.

Read stage units, up-axis, default prim, layer stack, load rules, and edit target before editing. `/World` and meter/Z-up are common conventions, not USD requirements. A missing prim can be inactive, unloaded, in another variant, behind a broken reference, or at a different path.

See also [usd.md](usd-composition-architecture.md) and [usd-pipeline.md](usd-pipeline.md).

## When to use

- Build new robot or environment assets.
- Restructure an existing asset with the Asset Transformer.
- Optimize RL training startup time and VRAM.
- Diagnose physics edits in a USDA that aren't taking effect.
- Debug joint limits, mass, or solver parameters.
- Create variants of an existing asset (configs, materials).

## Core Concept: One Binary Crate, Many USDA Layers

Isaac Sim's recommended asset structure splits a robot into one binary geometry crate plus a set of ASCII layers, composed by an `interface.usda`:

```
{robot}/
    interface.usda                         <- Final composed asset (entry point)
    payloads/
        base.usda                          <- Simulation-ready hierarchy + xforms
        geometries.usdc                    <- Mesh data ONLY (binary crate)
        instances.usda                     <- Mesh + material + collider assembly
        materials.usda                     <- Material defs (MDL bindings)
        Textures/                          <- Texture assets
        robot.usda                         <- Isaac robot schema + metadata
        Physics/
            physics.usda                   <- Neutral USD/Newton physics
            physx.usda                     <- PhysX-only tuning (sublayers physics.usda)
            mujoco.usda                    <- MuJoCo-only tuning (sublayers physics.usda)
```

USD `payload` arcs enable **lazy loading** — a payload is only loaded when explicitly requested. This is the key to RL startup optimization when appearance is not needed.

## File Format Decision Guide

The rule is simple: **binary crate (`.usdc`) for raw mesh data, USDA for everything else.** The Asset Transformer's `GeometriesRoutingRule` enforces this split automatically.

| Layer | Format | Why |
|---|---|---|
| `geometries.usdc` | `.usdc` (binary crate) | Mesh topology, points, indices — high-volume numeric data, never edited by hand |
| `base.usda` | `.usda` | Hierarchy and transforms — diffable, hand-editable |
| `instances.usda` | `.usda` | References meshes + applies materials + collision approximation choice |
| `materials.usda` | `.usda` | Material prims, MDL shader bindings — readable look-dev |
| `physics.usda` / `physx.usda` / `mujoco.usda` | `.usda` | Joint limits, masses, solver params — frequent tuning |
| `robot.usda` | `.usda` | Isaac robot schema metadata and relationships |
| `interface.usda` | `.usda` | Composition arcs (references, payloads, variants) — the entry point |
| Texture assets | original (PNG, JPG, EXR) + `.mdl` | Stored under `Textures/` |
| Archive/portable | `.usdz` | Single-file distribution (iOS AR) |

**Rationale:** Mesh arrays are large and never hand-edited, so binary crate wins on size and load time. Everything else is small, frequently inspected, and benefits from being diffable in version control and editable by both humans and agents.

## Producing This Structure: Asset Transformer

Use the Asset Transformer (Isaac Sim Structure profile) to convert an imported URDF/MJCF asset into the layout above. The relevant rules:

- `GeometriesRoutingRule` — extracts mesh prims to `geometries.usdc` (binary), creates instanceable references in `instances.usda`. Set `save_base_as_usda: true` to keep `base` ASCII.
- `MaterialsRoutingRule` — deduplicates materials into `materials.usda`, copies textures to `Textures/`.
- `SchemaRoutingRule` — splits physics, physx, mujoco, and robot schemas into their respective USDA layers.
- `InterfaceConnectionRule` — generates `interface.usda` with the composition arcs.

Refer to the Asset Transformer Rules Reference for the full pipeline.

## Physics USDA Schema

### physics.usda — Joint and Mass Definitions

```usda
#usda 1.0

def PhysicsRevoluteJoint "FL_hip_joint" {
    uniform token physics:axis = "X"
    float physics:lowerLimit = -46.0
    float physics:upperLimit = 46.0
    rel physics:body0 = </Robot/trunk>
    rel physics:body1 = </Robot/FL_hip>
}

def RigidBodyAPI "trunk" {
    float physics:mass = 4.713
    point3f physics:centerOfMass = (0.012, 0.002, -0.002)
    float3 physics:diagonalInertia = (0.0120, 0.0220, 0.0270)
}
```

### physx.usda — PhysX-Only Tuning

```usda
#usda 1.0

def PhysxJointAPI "FL_hip_joint" {
    float physxJoint:maxJointVelocity = 20.0
    float physxJoint:jointFriction = 0.05
}

def PhysxRigidBodyAPI "trunk" {
    bool physxRigidBody:enableGyroscopicForces = true
    float physxRigidBody:maxDepenetrationVelocity = 10.0
    int physxRigidBody:solverPositionIterationCount = 32
    int physxRigidBody:solverVelocityIterationCount = 1
}
```

`physx.usda` typically sublayers `physics.usda` so PhysX-only opinions stack on top of the neutral physics definition.

## RL optimization: skip appearance

The biggest RL training startup optimization: **don't load appearance payloads**.

When Isaac Lab loads a robot for RL:
1. Loads: `interface.usda` + base + physics layers (joints, masses, collision shapes).
2. Skips: `materials.usda` and `Textures/` (irrelevant for physics sim).

Lab `ArticulationCfg` belongs to isaac-lab-3. The pattern is: spawn the composed `interface.usda` and do not pull appearance payloads when physics-only.

Flattening does not embed external textures. Open the delivered entry point in a fresh context and inspect dependencies, bounds, materials, and required physics.

A point instancer is not an independently simulated articulation. Instance proxies are read-only: edit the source, override deliberately, or de-instance only the subtree that needs unique state.

## Composition arc precedence

LIVRPS (local, inherits, variants, references, payloads, specializes) is a debugging mnemonic. Sublayers order the **local** layer stack; they are not another LIVRPS slot.

Inspect `attribute.GetPropertyStack(Usd.TimeCode.Default())` and the active edit target for an actual conflict. References compose assets; payloads defer loading according to stage load rules.

If your physics USDA changes "don't take effect", check that the override opinion is in a higher-precedence arc/layer than the value you are trying to replace. Prefer a placement wrapper or deliberate override to clearing a referenced transform stack; `ClearXformOpOrder` can discard authored scale or orientation.

Use separate geometry, material, physics, and composition layers only when ownership or reuse needs them.

## Common debugging

```python
from pxr import Usd

attr = prim.GetAttribute("physics:mass")
for spec in attr.GetPropertyStack(Usd.TimeCode.Default()):
    print(f"  Layer: {spec.layer.GetDisplayName()} = {spec.default}")
```
