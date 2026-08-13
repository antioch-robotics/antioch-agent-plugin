# Physics authoring — Isaac Sim 6.0.1

Detail reference for the `isaac-sim-6` skill. Both backends (PhysX, Newton)
share `UsdPhysics.*`; backend-specific behavior is called out per section.
All `pxr`/`isaacsim` imports belong inside function bodies (the skill's Rule
#1) — snippets here assume they run inside a function.

## Contents

- [Backend selection](#backend-selection)
- [PhysicsScene setup](#physicsscene-setup) — Hz table, solver iterations,
  stabilization
- [Per-prim setup](#per-prim-setup) — colliders, compound bodies, scale,
  mass & inertia
- [Contact materials](#contact-materials)
- [Joint drives](#joint-drives)
- [Mechanisms](#mechanisms) — conveyor surfaces, doors, turntables
- [Newton vs PhysX](#newton-vs-physx)
- [Deformables](physics-deformables.md)
- [Physics sensors](#physics-sensors) — contact/IMU, raycasts
- [Physics-to-USD readback](#physics-to-usd-readback-critical)
- [Observed limits from NVIDIA's benchmarking](#observed-limits-from-nvidias-benchmarking)

## Backend selection

`isaacsim.core.simulation_manager` owns engine registration. Antioch boots
PhysX by default. Select Newton in the boot profile so its extensions and
auto-switch setting are present before the first stage opens:

```python
import antioch

antioch.boot(physics_engine="newton")
```

For a native engine check, always inspect the active backend:

```python
from isaacsim.core.simulation_manager import SimulationManager

# `antioch.boot(physics_engine="newton")` already switched the backend.
print(SimulationManager.get_active_physics_engine())
```

Calling the native switch without first booting the Newton extensions reports
`Engine 'newton' not found. Available: PhysX`. Restart with
`antioch.boot(physics_engine="newton")`; a second boot cannot change the
profile. The first Newton workload can compile Warp/Newton CUDA kernels for
about two minutes on a fresh machine. Later runs reuse the engine cache.

Some Isaac Sim 6.0.1 Newton starts also print a `Detach stage` warning while
the default PhysX stage is released. Treat it as expected only when the active
engine check reports `newton` and the run advances. If Newton is not active or
the run stops, keep the warning as upstream evidence instead of suppressing it.

When the Newton extension is enabled, its USD integration exposes
`NewtonConfig` (per-sim settings), `XPBDSolverConfig`, `MuJoCoSolverConfig`,
and `tensors` (NumPy / PyTorch / Warp frontends) under
`isaacsim.physics.newton`. That extension namespace is not the runtime solver
import: the pinned softpack image raised `ModuleNotFoundError` for it, while
the top-level `newton` package worked. Newton schemas are authored as
`newton:*` attributes via the `newton_usd_schemas` plugin; the extension
resolves them through `newton._src.usd.schemas`. Most `PhysxSchema.*`
attributes are still honored. See `references/physics-deformables.md` for the
two-layer distinction and the live CUDA boundary.

Choosing between them:

| | PhysX (default) | Newton (experimental in 6.0) |
|---|---|---|
| Parallel envs | proven to ~4,096 | scales past 10K |
| GPU | single | multi-GPU capable |
| Feature coverage | everything (deformables, surface grippers, material randomization) | Newton ModelBuilder supports particles/soft grids/cloth; USD integration remains narrower and several PhysX features are not ported |
| USD integration | `PhysxSchema.*` | `newton:*` schema; honors most PhysX attrs |

Pick PhysX for legacy scenes and feature coverage, Newton for large-scale RL
batches — and verify the active engine instead of assuming.

## PhysicsScene setup

```python
from pxr import UsdPhysics, PhysxSchema, Gf

ps = UsdPhysics.Scene.Define(stage, "/World/PhysicsScene")
ps.CreateGravityDirectionAttr().Set(Gf.Vec3f(0, 0, -1))
ps.CreateGravityMagnitudeAttr().Set(9.81)

px = PhysxSchema.PhysxSceneAPI.Apply(ps.GetPrim())
px.CreateTimeStepsPerSecondAttr().Set(240)
px.CreateEnableCCDAttr().Set(True)
px.CreateEnableStabilizationAttr().Set(True)
px.CreateSolverTypeAttr().Set("TGS")  # TGS preferred for articulations
```

Define the PhysicsScene before the first `timeline.play()` — first-play hangs
usually trace to a missing scene.

### Physics Hz

| Scenario | Hz | Notes |
|---|---|---|
| Standard rigid bodies | 60–120 | warehouse, general sim |
| Stacking / contact-rich | 240 | tight contact resolution |
| High-velocity impacts | 120 + 2–4 substeps | pair with CCD |
| Small-part vibration | >= 4x vibration freq (typ. 480) | resolve oscillation |
| Spinning bodies / gyros | 480 | angular momentum precision |
| Stiff contact chains | 480 | solver needs sub-iterations |

Rule of thumb: timestep must resolve > 4x the highest frequency in the system.

### Solver iteration counts (per body, on `PhysxRigidBodyAPI`)

| Scenario | Position iters | Velocity iters |
|---|---|---|
| Simple tumbling bodies | 16 | 4 |
| Stacking | 32 | 8 |
| Complex articulations | 64 | 16 |
| Stiff contact chains | 64 | 32 |

```python
pxrb = PhysxSchema.PhysxRigidBodyAPI.Apply(prim)
pxrb.CreateSolverPositionIterationCountAttr().Set(32)
pxrb.CreateSolverVelocityIterationCountAttr().Set(8)
pxrb.CreateEnableCCDAttr().Set(True)
```

### Stabilization

`EnableStabilization` is on by default and helps stacks settle — but it
destroys angular momentum on free-spinning bodies. Disable it for tops, gyros,
flywheels, pendulums, anything whose correctness depends on conserved angular
velocity.

## Per-prim setup

- `CollisionAPI` alone = static collider (terrain, walls).
- Simple dynamic body: `RigidBodyAPI` + `CollisionAPI` on the same prim.
- Compound body (multi-part link, gripper, chassis): `RigidBodyAPI` on the
  parent prim, `CollisionAPI` on each child mesh — USD assigns every collider
  to the nearest ancestor carrying a rigid-body API, so children compose into
  one body. This is the standard compound-collider layout, not a bug. If a
  compound body shows intermittent contact loss, hoisting the collider onto
  the body prim is the classic remedy — check it before deeper debugging.
- Kinematic body (conveyors, scripted motion): `RigidBodyAPI` +
  `CreateKinematicEnabledAttr().Set(True)` + `CollisionAPI`.
- Floating-base robot: the articulation root belongs on the **parent Xform**,
  not the root rigid body — misplacing it fails at reset with the opaque
  `Failed to get root link transforms from backend`.

What to avoid is ambiguity: a collider with NO rigid-body ancestor is static,
and a rigid body with no collider anywhere under it falls through the world.
When contacts misbehave, check which prim actually carries each API.

Kinematic→dynamic transitions mid-scene are unreliable in a full robot scene:
toggling bodies dynamic after robot trajectories produced large displacements
and non-finite PhysX bounds (`Illegal BroadPhaseUpdateData - non-finite
bounds`) even though the identical bodies were stable in an isolated probe.
Design settle checks to run on bodies that were dynamic from the start, or
report kinematic-settle results as such.

### Scaling colliders

Scaling a collider prim is not forbidden — NVIDIA's own 6.0.1 hello-world
tutorial scales a cube and then applies `CollisionAPI`, and PhysX bakes
uniform scale into simple geometry correctly. The real caveats:

- Nonuniform or time-varying scale can produce wrong bounds for certain
  collision approximations (convex hull, SDF, mesh) — prefer authoring the
  size on the shape itself (`Cube.size`, `Cylinder.radius`) when you can, or
  re-run the collision approximation after scaling.
- When scaled-collider bounds look wrong, fall back to the translate-first
  pattern: an unscaled parent xform carries the placement and the child mesh
  stays at scale 1, so PhysX never sees a scaled collider.
- When in doubt, enable collision-shape visualization and inspect the drawn
  bounds instead of assuming.

```python
mesh = UsdGeom.Cube.Define(stage, "/World/Ground/Mesh")
mesh.CreateSizeAttr().Set(2.0)
UsdGeom.Xformable(mesh.GetPrim()).AddScaleOp().Set(Gf.Vec3f(25.0, 25.0, 0.05))
UsdPhysics.CollisionAPI.Apply(mesh.GetPrim())
```

`Cube.size=1.0` = half-extent 0.5. Use `size=2.0` when you want the scale op
to equal the half-extent.

### Mass & inertia

```python
mass_api = UsdPhysics.MassAPI.Apply(prim)
mass_api.CreateMassAttr().Set(0.25)  # kg
mass_api.CreateCenterOfMassAttr().Set(Gf.Vec3f(0, 0, 0.05))  # local
mass_api.CreateDiagonalInertiaAttr().Set(Gf.Vec3f(1e-4, 1e-4, 2e-4))  # kg·m²
```

## Contact materials

```python
def create_contact_material(stage, path, static_friction=0.5, dynamic_friction=0.4, restitution=0.1):
    from pxr import UsdPhysics, UsdShade

    material = UsdShade.Material.Define(stage, path)
    mat = UsdPhysics.MaterialAPI.Apply(material.GetPrim())
    mat.CreateStaticFrictionAttr().Set(static_friction)
    mat.CreateDynamicFrictionAttr().Set(dynamic_friction)
    mat.CreateRestitutionAttr().Set(restitution)
    return material


def bind_contact_material(material, collider_prim) -> None:
    from pxr import UsdShade

    UsdShade.MaterialBindingAPI.Apply(collider_prim).Bind(material, materialPurpose="physics")
```

Materials must be **bound, not applied**: `UsdPhysics.MaterialAPI.Apply(collider_prim)` on the collider itself silently does nothing — PhysX reads only a bound material prim, through the material binding. And the default `frictionCombineMode` is `average`, so a 0.0-friction material against a 1.0-friction floor yields 0.5 — a "frictionless" caster commanded at 1.0 rad/s spun at 0.28 rad/s until the material was bound correctly. For a genuinely frictionless contact, set `frictionCombineMode=min` on `PhysxMaterialAPI`.

| Material pairing | Static μ | Dynamic μ | Restitution |
|---|---|---|---|
| Concrete on concrete | 0.6 | 0.5 | 0.05 |
| Steel on steel | 0.74 | 0.57 | 0.6 |
| Rubber on rubber | 0.8 | 0.7 | 0.5 |
| Rubber on concrete | 1.0 | 0.8 | 0.3 |
| Wood on wood | 0.5 | 0.3 | 0.2 |
| Metal generic | 0.4 | 0.3 | 0.2 |
| Plastic | 0.4 | 0.3 | 0.3 |

For stiff contact chains, set `restitutionCombineMode=max` on
`PhysxMaterialAPI` so the highest restitution wins per contact.

## Joint drives

```python
drive = UsdPhysics.DriveAPI.Apply(joint, "angular")  # "angular" | "linear"
drive.CreateTypeAttr().Set("force")  # "force" | "acceleration"
drive.CreateStiffnessAttr().Set(1000.0)  # Kp (Nm/rad)
drive.CreateDampingAttr().Set(100.0)  # Kd (Nm·s/rad)
drive.CreateMaxForceAttr().Set(500.0)
drive.CreateTargetPositionAttr().Set(0.0)
```

- RL training commands torques directly: zero stiffness and damping (or don't
  author a DriveAPI) so no active PD drive fights the RL agent.
- Revolute pendulum joints: zero the joint friction via
  `PhysxSchema.PhysxJointAPI.CreateJointFrictionAttr().Set(0.0)`.

## Mechanisms

### Conveyor belt

A conveyor is normally a **kinematic rigid body with a surface velocity**, not
a dynamic body that is teleported every frame. The pinned helper applies the
required `RigidBodyAPI`, `CollisionAPI`, and `PhysxSurfaceVelocityAPI` and
builds the velocity action graph:

```python
def add_conveyor(stage, prim_path: str):
    from isaacsim.asset.gen.conveyor import create_conveyor_belt

    conveyor = stage.GetPrimAtPath(prim_path)
    return create_conveyor_belt(stage, conveyor)
```

For a small hand-authored belt, make the same contract explicit and set the
surface velocity in the belt's local frame (or set the local-space attribute
to false when the vector is world-space):

```python
def make_kinematic_belt(prim) -> None:
    from pxr import Gf, PhysxSchema, UsdPhysics

    rigid = UsdPhysics.RigidBodyAPI.Apply(prim)
    UsdPhysics.CollisionAPI.Apply(prim)
    rigid.CreateKinematicEnabledAttr().Set(True)
    surface = PhysxSchema.PhysxSurfaceVelocityAPI.Apply(prim)
    surface.CreateSurfaceVelocityAttr().Set(Gf.Vec3f(0.5, 0.0, 0.0))
    surface.CreateSurfaceVelocityEnabledAttr().Set(True)
    surface.CreateSurfaceVelocityLocalSpaceAttr().Set(True)
```

The helper's public signature is `create_conveyor_belt(stage,
conveyor_prim, prim_name="ConveyorBeltGraph")`; inspect the generated graph
variable when changing speed. An animated prim plus contact can model a belt
whose geometry visibly travels, but keep collision enabled and validate the
contact direction; do not combine a dynamic drive with a kinematic surface
without measuring the resulting forces.

### Door or turntable

Use a `UsdPhysics.RevoluteJoint` between a fixed frame (`body0`) and the moving
part (`body1`), then drive the joint's angular degree of freedom. The same
pattern covers a door hinge and a turntable's vertical spindle:

```python
def add_revolute_drive(stage, body0_path: str, body1_path: str, joint_path: str) -> None:
    from pxr import Sdf, UsdPhysics

    joint = UsdPhysics.RevoluteJoint.Define(stage, joint_path)
    joint.CreateBody0Rel().SetTargets([Sdf.Path(body0_path)])
    joint.CreateBody1Rel().SetTargets([Sdf.Path(body1_path)])
    joint.CreateAxisAttr().Set("Z")
    drive = UsdPhysics.DriveAPI.Apply(joint.GetPrim(), "angular")
    drive.CreateTypeAttr().Set("force")
    drive.CreateStiffnessAttr().Set(1000.0)
    drive.CreateDampingAttr().Set(100.0)
    drive.CreateMaxForceAttr().Set(500.0)
    drive.CreateTargetPositionAttr().Set(90.0)
```

Use a door's hinge axis for `Z` (or the authored axis), and a vertical axis for
a turntable. Tune stiffness, damping, and target units against the composed
asset; a drive is an actuator, not a replacement for correct body links,
limits, collision shapes, and masses.

## Newton vs PhysX

| You want | Use |
|---|---|
| RL at thousands of envs | Newton (Featherstone or MuJoCo) |
| Differentiable simulation | Newton (Featherstone or SemiImplicit — the only differentiable solvers, "basic" support) |
| Newton ModelBuilder soft grid / cloth | top-level `newton.ModelBuilder.add_soft_grid` + `newton.solvers.SolverVBD`; see `physics-deformables.md` |
| USD-schema deformables | Isaac Sim's PhysX deformable layer (`DeformableBodyAPI` / experimental deformable prims), not the Newton builder API |
| MuJoCo-validated baselines | Newton SolverMuJoCo |
| Legacy PhysX scene from Isaac Sim 5.x | PhysX |

| Solver | Coordinates | Differentiable | Best for |
|---|---|---|---|
| SolverFeatherstone | Generalized | Basic | articulated robots (default); also particles, soft bodies, cloth without self-collision |
| SolverMuJoCo | Generalized | No | rigid bodies + articulations — locomotion, MuJoCo policy ports; no particles/cloth/soft bodies |
| SolverXPBD | Maximal | No | cloth (no self-collision), experimental soft bodies |
| SolverSemiImplicit | Maximal | Basic | fast prototyping; coverage similar to Featherstone |
| SolverVBD | Per-vertex (implicit) | No | Newton ModelBuilder particles and soft grids, cloth, rigid bodies, limited-joint articulations |

The older capability wording that says VBD has “no soft bodies” describes a
different USD-schema/integration layer. It does not contradict the pinned
Newton builder surface: `ModelBuilder.add_soft_grid(...)` plus `SolverVBD`
demonstrably deforms a soft grid. The integration-layer distinction and the
CUDA hazards are documented in `references/physics-deformables.md`.

Critical init order: never `import torch` before Newton physics settles —
CUDA context conflict hangs Kit. Defer torch until after `timeline.play()` +
a settle loop; use `map_location="cpu"` for policy inference if VRAM is tight.

## Physics sensors

Namespace: `isaacsim.sensors.experimental.physics` (legacy
`isaacsim.sensors.physics` is deprecated). All require
`app_utils.play(commit=True)` before `get_data()`; do not call the legacy
sensor `initialize()` flow.

```python
from isaacsim.sensors.experimental.physics import Contact, ContactSensor
import isaacsim.core.experimental.utils.app as app_utils

contact = Contact.create(path="/World/Robot/foot/contact", min_threshold=0.0, max_threshold=1e6, radius=-1)
sensor = ContactSensor(contact)
app_utils.play(commit=True)
reading = sensor.get_data()
```

The experimental contact stub defines a negative radius as **no radius
filtering** and uses `-1` as its default, so this gloss is stub-proven. It is
not an infinite sensor volume: thresholds, the sensor's body ancestor, and
the authored contact geometry still determine readings.

Same pattern for `IMU`/`IMUSensor` (frame keys: `linear_acceleration`,
`angular_velocity`, `orientation`). Runtime-only wrappers: `EffortSensor`
takes one joint path formatted `articulation_path/joint_name`;
`JointStateSensor` takes the articulation root prim path, not a joint path.

### Raycasts

The pinned stubs expose `raycast_closest` on the PhysX scene-query interfaces
(`omni.physics.core` and `omni.physx`), but not on
`omni.physics.tensors.SimulationView` or its valid `numpy`, `torch`,
`tensorflow`, and `warp` frontends. This is a stub-proven interface boundary,
not a claim that no other extension can add a raycast API. Use a scene query
for one ray:

```python
import carb
import omni.physx

hits = omni.physx.get_physx_scene_query_interface().raycast_closest(
    origin=carb.Float3(0.0, 0.0, 2.0),
    dir=carb.Float3(0.0, 0.0, -1.0),  # unit direction
    distance=100.0,
)
# hits: dict with 'position', 'normal', 'distance', 'collision', 'rigidBody', ...
```

For many parallel rays per step, use the batched ray-caster sensors instead:
`Raycast` / `RaycastSensor` in `isaacsim.sensors.experimental.physics`
author an `IsaacRaycastSensor` prim and read back all rays per step. RTX
sensors (camera, LiDAR) live under `isaacsim.sensors.experimental.rtx`.

## Physics-to-USD readback (critical)

Which accessor to read from: see SKILL.md's "Reading state — the XformCache
trap" table. The mechanism behind it: `updateToUsd=True` writes physics to
Fabric, not the USD layer, so `XformCache` returns initial poses forever
during sim.

```python
from isaacsim.core.experimental.prims import RigidPrim

rp = RigidPrim(paths="/World/Dice/Die_*")
app_utils.play(commit=True)
pos_wp, quat_wp = rp.get_world_poses()  # warp arrays, [w,x,y,z] quats
positions = pos_wp.numpy()  # (N, 3)
```

`Articulation` (same module) adds `get_dof_positions()`, `get_dof_velocities()`,
`get_jacobian_matrices()`; it needs the timeline playing for tensor data.

## Observed limits from NVIDIA's benchmarking

These come out of NVIDIA's own physics stress benchmarks on specific scenes —
treat them as starting points for suspicion, not universal laws, and
re-measure in your scene before designing around one.

- PhysX struggled to resolve sequential momentum transfer in same-island
  contact chains (Newton's cradle symptom: spheres oscillated in unison
  instead of transferring momentum). TGS/PGS, Hz, iterations, restitution,
  and CCD did not fix it in the benchmark — scripting the analytical transfer
  or switching backends did.
- Compound colliders tunneled in a benchmark spinning above ~50 rad/s for
  5–6 s. If you see this, try simpler convex hulls or higher physics Hz.
- Pure PhysX contact became unreliable in a benchmark with features below
  ~10 mm — for sub-centimeter mechanisms (escapements, ratchets) consider a
  hybrid kinematic/scripted approach and validate contact behavior directly.
- Benchmark revolute joints bled energy over long free swings (~40% over
  15 s) even with stabilization, damping, and friction zeroed. If exact
  period matters in your scene, measure the decay and decide whether it
  clears your tolerance.
