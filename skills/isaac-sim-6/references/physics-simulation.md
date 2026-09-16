# Physics Simulation in Isaac Sim

Adapted from NVIDIA [`physics-simulation/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/physics-simulation/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Configure PhysicsScene and per-prim rigid bodies, collisions, materials, joint drives, solver selection, and physics sensors with worked examples.

Startup and engine pinning are in [../SKILL.md](../SKILL.md). Scene composition is in [usd.md](usd-composition-architecture.md) / [usd-composition-architecture.md](usd-composition-architecture.md). Articulations: [usd-articulation.md](usd-articulation.md). Importers: [urdf-mjcf-to-usd-conversion.md](urdf-mjcf-to-usd-conversion.md).

## Upstream helpers (source examples)

| Script | Purpose |
|---|---|
| [`scripts/prim_physics_setup.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/physics-simulation/scripts/prim_physics_setup.py) | Per-prim physics setup helpers for Isaac Sim / USD (Kit 110) |

Not a bundled command.

Targets Isaac Sim 6.0.1 / Kit 110.1.2. Both backends share `UsdPhysics.*`; backend-specific behavior is called out per section.

## Backend selection (Kit 110)

Antioch defaults to **PhysX**. Select Newton with `SimulationConfig(physics_engine="newton")` **before** startup and verify `SimulationManager.get_active_physics_engine()`. Enabling `isaacsim.physics.newton` is not a promise that an otherwise PhysX scene supports the same features.

When Antioch selects Newton it enables `isaacsim.physics.newton` / `isaacsim.physics.newton.tensors` and sets `auto_switch_on_startup=true`. That is not the default for an Antioch PhysX session. Do not assume the upstream `isaacsim.exp.full.kit` auto-switch applies here.

PhysX, Isaac's Newton USD/tensor integration, and the standalone `newton` solver are separate surfaces.

```python
from isaacsim.core.simulation_manager import SimulationManager

print(SimulationManager.get_active_physics_engine())
print(SimulationManager.get_available_physics_engines())
```

Switching after the stage is open is possible (`SimulationManager.switch_physics_engine("newton")`) but startup selection is the supported Antioch path. `isaacsim.physics.newton.get_available_physics_engines` / `get_active_physics_engine` exist on the Newton extension; prefer `SimulationManager` so the call works when Newton is not enabled.

Newton config classes (extension Python API):

| Class | Role |
|---|---|
| `isaacsim.physics.newton.NewtonConfig` | per-sim settings (CUDA graph capture, fabric sync, contact/joint defaults) |
| `XPBDSolverConfig` | XPBD solver (rigid + soft) |
| `MuJoCoSolverConfig` | MuJoCo Warp solver |
| `isaacsim.physics.newton.tensors` | NumPy / PyTorch / Warp frontends |

Both Newton and PhysX consume the standard `UsdPhysics.Scene` + `PhysxSchema.PhysxSceneAPI`; many `PhysxSchema.*` attributes are still honored under Newton, plus Newton reads its solver config via `omni.usd.schema.newton`.

Use one stepping owner: classic `World`/`SimulationContext`, experimental `SimulationManager`, or a Lab environment loop. Do not wrap a framework that already owns an app in another `SimulationApp` or a second `World`.

## Stack and reading order

1. This reference: scene config, per-prim setup, contact materials, drives, sensors, readback, backend selection.
2. [usd-articulation.md](usd-articulation.md): multi-link articulations + Robot Schema overlay.
3. [urdf-mjcf-to-usd-conversion.md](urdf-mjcf-to-usd-conversion.md): importer config (RL vs teleop drives).
4. [isaac-sim-troubleshooting.md](isaac-sim-troubleshooting.md): when physics misbehaves.

Mechanism recipes (impact, feeders, dominoes, tops, cradles, pendulum waves, escapements) live in [physics-simulation-examples.md](physics-simulation-examples.md).

---

## Part 1 — Scene-level configuration

### PhysicsScene setup

A physics scene is required, but `/World/PhysicsScene` is only a convention; do not create a conflicting second scene. Read stage units and up-axis.

```python
from pxr import Usd, UsdGeom, UsdPhysics, PhysxSchema, Gf

ps = UsdPhysics.Scene.Define(stage, "/World/PhysicsScene")
ps.CreateGravityDirectionAttr().Set(Gf.Vec3f(0, 0, -1))
ps.CreateGravityMagnitudeAttr().Set(9.81)

px = PhysxSchema.PhysxSceneAPI.Apply(ps.GetPrim())
px.CreateTimeStepsPerSecondAttr().Set(240)  # see Hz table below
px.CreateEnableCCDAttr().Set(True)
px.CreateEnableStabilizationAttr().Set(True)
px.CreateSolverTypeAttr().Set("TGS")  # TGS or PGS; TGS preferred for articulations
```

### Physics Hz selection (starting points)

| Scenario | Hz | Notes |
|---|---|---|
| Standard rigid-body scenes | 60–120 | Default for warehouse, general sim |
| Stacking / contact-rich | 240 | Tight contact resolution |
| High-velocity impacts | 120 with 2–4 substeps | Pair with CCD |
| Small-part vibration (feeders) | ≥ 4× vibration freq, typically 480 | Resolve oscillation correctly |
| Spinning bodies / gyros | 480 | Numerical precision for angular momentum |
| Stiff contact chains (cradles, escapements) | 480 | Solver needs many sub-iterations |

**Rule of thumb:** physics timestep should resolve the highest frequency in the system (vibration, spin, contact-stiffness mode); 4× that frequency is a common starting point, not a proof of convergence. Compare the claimed result under a smaller timestep or changed solver budget when numerical convergence matters.

With `physics_dt` smaller than `render_dt`, `World.step(render=True)` advances `render_dt / physics_dt` physics steps and `render=False` advances one. Count simulated time from `run.sim_s` or physics callbacks, not loop calls.

### Solver iteration counts (per-body, starting points)

Set on `PhysxRigidBodyAPI` per body that needs it. Higher = more accurate, slower.

| Scenario | Position iters | Velocity iters |
|---|---|---|
| Simple rigid bodies, tumbling | 16 | 4 |
| Stacking | 32 | 8 |
| Complex joints / articulations | 64 | 16 |
| Stiff contact chains (cradle, escapement) | 64 | 32 |

```python
pxrb = PhysxSchema.PhysxRigidBodyAPI.Apply(prim)
pxrb.CreateSolverPositionIterationCountAttr().Set(32)
pxrb.CreateSolverVelocityIterationCountAttr().Set(8)
pxrb.CreateEnableCCDAttr().Set(True)
```

### When to disable stabilization

`EnableStabilizationAttr` is on by default and helps stacks settle. It **destroys angular momentum** on free-spinning bodies. Disable it for:
- Spinning tops, gyros, flywheels
- Pendulum mechanisms (clock escapements, pendulum waves)
- Anything whose correctness depends on conserved angular velocity

```python
px.CreateEnableStabilizationAttr().Set(False)
```

---

## Part 2 — Per-prim physics setup

### RigidBody / Collision / Static / Kinematic

Use [`scripts/prim_physics_setup.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/physics-simulation/scripts/prim_physics_setup.py) for dynamic, static, and kinematic body setup. It keeps the `RigidBodyAPI`, `MassAPI`, and `CollisionAPI` application sequence in one executable implementation.

Keep `UsdPhysics.RigidBodyAPI` and colliders on the intended ownership hierarchy; an unintended nested rigid body changes it. Visual versus collision geometry, static/dynamic/kinematic mode, mass, center of mass, inertia, initial penetration, filters, self-collision, materials, friction, restitution, contact offsets, joint frames/limits, actuator mode, stiffness, damping, and effort limits all matter. A convex hull can close a visible gap, and nonuniform scale changes collision behavior.

Collider children can belong to a rigid-body ancestor. Keep that ownership
hierarchy intact; do not add nested rigid bodies merely to put both APIs on
the same prim.

### Static colliders with scale — translate-first pattern

A parent xform for position and a child mesh for scale keeps the authored
transform order clear. Inspect the resulting collision bounds after edits:

```python
# CORRECT
xf = UsdGeom.Xform.Define(stage, "/World/Ground")
UsdGeom.Xformable(xf.GetPrim()).AddTranslateOp().Set(Gf.Vec3d(0, 0, -0.05))
mesh = UsdGeom.Cube.Define(stage, "/World/Ground/Mesh")
mesh.CreateSizeAttr().Set(1.0)
UsdGeom.Xformable(mesh.GetPrim()).AddScaleOp().Set(Gf.Vec3f(50.0, 50.0, 0.1))
UsdPhysics.CollisionAPI.Apply(mesh.GetPrim())
```

**`Cube.size=1.0`** means the cube has half-extents of 0.5, not 1.0. Use `size=2.0` when you want "the scale op equals the half-extent."

### Mass and inertia

```python
mass_api = UsdPhysics.MassAPI.Apply(prim)
mass_api.CreateMassAttr().Set(0.25)  # kg
mass_api.CreateCenterOfMassAttr().Set(Gf.Vec3f(0, 0, 0.05))  # local
mass_api.CreateDiagonalInertiaAttr().Set(Gf.Vec3f(1e-4, 1e-4, 2e-4))  # kg·m²
```

For URDF-imported robots, prefer `import_inertia_tensor: true` in Lab `config.yaml` over auto-computed geometric inertia (see [urdf-mjcf-to-usd-conversion.md](urdf-mjcf-to-usd-conversion.md)).

---

## Part 3 — Contact materials

```python
def create_contact_material(stage, mat_path, static_friction=0.5, dynamic_friction=0.4, restitution=0.1):
    prim = stage.DefinePrim(mat_path)
    mat = UsdPhysics.MaterialAPI.Apply(prim)
    mat.CreateStaticFrictionAttr().Set(static_friction)
    mat.CreateDynamicFrictionAttr().Set(dynamic_friction)
    mat.CreateRestitutionAttr().Set(restitution)
    return mat
```

### Reference values (tuning starting points, not universal laws)

| Material pairing | Static μ | Dynamic μ | Restitution |
|---|---|---|---|
| Concrete on concrete | 0.6 | 0.5 | 0.05 |
| Steel on steel | 0.74 | 0.57 | 0.6 |
| Rubber on rubber | 0.8 | 0.7 | 0.5 |
| Rubber on concrete | 1.0 | 0.8 | 0.3 |
| Wood on wood | 0.5 | 0.3 | 0.2 |
| Metal generic | 0.4 | 0.3 | 0.2 |
| Plastic (dice) | 0.4 | 0.3 | 0.3 |
| Felt (casino) | 0.5 | 0.4 | 0.2 |
| Cardboard on steel | 0.4 | 0.3 | 0.1 |

For chains of stiff contacts (Newton's cradle, escapements), set `restitutionCombineMode=max` on `PhysxMaterialAPI` so the highest restitution wins at each contact.

---

## Part 4 — Joint drives

```python
joint = stage.GetPrimAtPath("/World/Robot/joint_arm")
drive = UsdPhysics.DriveAPI.Apply(joint, "angular")  # "angular" | "linear"
drive.CreateTypeAttr().Set("force")  # "force" | "acceleration"
drive.CreateStiffnessAttr().Set(1000.0)  # example Kp (Nm/rad for angular)
drive.CreateDampingAttr().Set(100.0)  # example Kd (Nm·s/rad)
drive.CreateMaxForceAttr().Set(500.0)  # torque/force limit
drive.CreateTargetPositionAttr().Set(0.0)  # target (deg or m)
```

A drive target is not a direct state write. Author initial conditions before reset; initialize views/controllers through the selected framework, reset, then apply live targets.

**For RL training**, the agent commands torques directly. Set drive_type to `none` and stiffness/damping to 0 in `config.yaml` (see [urdf-mjcf-to-usd-conversion.md](urdf-mjcf-to-usd-conversion.md)). Active PD drives fight the RL agent.

**For revolute pendulum joints** (clock escapements, pendulum waves), set joint friction to 0 when you need conserved swing:

```python
joint_api = PhysxSchema.PhysxJointAPI.Apply(joint)
joint_api.CreateJointFrictionAttr().Set(0.0)
```

---

## Part 5 — Backend selection (Newton vs PhysX)

### Quick choice

| You want | Use |
|---|---|
| Default Antioch session, legacy PhysX scenes | **PhysX** |
| RL training with thousands of envs | **Newton** (Featherstone or MuJoCo) — explicit `SimulationConfig` |
| Differentiable simulation | **Newton** (standalone vs Isaac integration differ) |
| Soft bodies, cloth, deformables | **Newton** (VBD or XPBD) — verify the pinned feature |
| Validated against MuJoCo baselines | **Newton SolverMuJoCo** |

Retrieve the selected backend's implementation and test that path. Enabling Newton does not imply feature parity with PhysX.

### Newton solvers

| Solver | Coordinates | Differentiable | Best For |
|---|---|---|---|
| **SolverFeatherstone** | Generalized | Yes (Warp) | Articulated robots (default for manipulators, legged) |
| **SolverMuJoCo** | Generalized | Yes (mujoco-warp) | Validated locomotion, MuJoCo policy ports |
| **SolverXPBD** | Maximal | Partial | Soft constraints, cables, ropes |
| **SolverSemiImplicit** | Maximal | Yes (Warp) | Fast prototyping, simple rigid bodies |
| **SolverVBD** | (deformable) | Yes | Soft bodies, deformables |

These solver names are Newton-library surfaces. Confirm which are wired through Isaac's USD/tensor integration at this pin before depending on them.

### Newton vs PhysX differences

| Aspect | PhysX | Newton |
|---|---|---|
| Backend | Closed C++/CUDA | Warp/CUDA (open, JIT) |
| Coordinates | Maximal (6DoF per body) | Generalized (Featherstone) or maximal |
| Differentiable | No | Yes (native Warp autodiff) |
| Multi-GPU | Limited | Yes (Warp device abstraction) |
| USD integration | Schema extensions | Native USD loader |
| Performance ceiling | Good < 4096 envs | Designed for 10K+ envs |

### Newton + Torch — init order

For a Newton/Torch startup hang, isolate backend initialization and device allocation
in a small reproduction. Follow the selected framework's import and startup
order; do not impose an arbitrary settle loop as a dependency. A CPU checkpoint
load does not make a controller's inference path CPU-compatible.

### Newton-specific configuration (Isaac Lab)

```yaml
# config.yaml for URDF→USD conversion (Isaac Lab)
make_instanceable: true    # useful when instances share immutable geometry
fix_base: false            # true for fixed-base arm; false for mobile/legged
```

See [urdf-mjcf-to-usd-conversion.md](urdf-mjcf-to-usd-conversion.md) for the full schema. Lab convert scripts belong to isaac-lab-3; run conversion in the configured remote service.

---

## Part 6 — Physics sensors

The current namespace is `isaacsim.sensors.experimental.physics` (authoring + runtime classes paired). Enable `isaacsim.sensors.experimental.physics` in `SimulationConfig.extensions`.

Classic `isaacsim.sensors.physics` is **deprecated, not removed**. Classic contact additionally requires the immediate parent to carry `UsdPhysics.CollisionAPI`. Enable its owning extension `isaacsim.sensors.physics` in `SimulationConfig.extensions` before startup; the shipped classic module is not enabled by default. Do not treat a Python import path as the extension ID.

> **Migration:** see [Migrating from `isaacsim.sensors.physics` to `isaacsim.sensors.experimental.physics`](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_6_0/sensors_physics_to_experimental_physics.html#isaacsim-sensors-physics-migration).

Author experimental contact at a child path such as `/World/Robot/foot/contact_sensor`, not at the collider/body path itself. `Contact.create` needs an enabled rigid-body ancestor and collision geometry.

Proximity, a command acknowledgement, or a missing contact report is not physical contact. Check filters, reporting thresholds, force/impulse units, and sample time. A contact value read over a render step is accumulated over `render_dt / physics_dt` substeps; divide by that interval, not by `physics_dt`.

### Contact

```python
from isaacsim.sensors.experimental.physics import Contact, ContactSensor
import isaacsim.core.experimental.utils.app as app_utils

contact = Contact.create(
    "/World/Robot/foot/contact_sensor",
    min_threshold=0.0,
    max_threshold=1e6,
    radius=-1,  # -1 = use collision shape
)
sensor = ContactSensor(contact)
app_utils.play(commit=True)  # required before get_data()
reading = sensor.get_sensor_reading()  # raw reading supplies is_valid
frame = sensor.get_data()  # dictionary supplies time and contact data
```

### IMU

```python
from isaacsim.sensors.experimental.physics import IMU, IMUSensor

imu = IMU.create("/World/Robot/body/imu", linear_acceleration_filter_size=5)
sensor = IMUSensor(imu)
app_utils.play(commit=True)
reading = sensor.get_sensor_reading(read_gravity=True)
frame = sensor.get_data(read_gravity=True)  # dictionary; orientation is WXYZ
```

Step the owning framework until `reading.is_valid` and its timestamp advances,
with a bounded deadline. `get_data()` can retain a previous frame when the
current IMU reading is invalid. Filter sizes are sample counts, not rates.

### Effort / joint state

Runtime-only classes from the same module: effort targets one joint;
joint state targets the actual articulation root.

```python
from isaacsim.sensors.experimental.physics import EffortSensor, JointStateSensor

effort = EffortSensor("/World/Robot/joint_arm_1")
joint = JointStateSensor("/World/Robot")
app_utils.play(commit=True)
reading = effort.get_data()  # dictionary; check is_valid and time
state = joint.get_data()  # dictionary, including dof_names and positions
```

### Raycast (scene query)

```python
from omni.physx.scripts.ifaces import get_physx_scene_query_interface

hit = get_physx_scene_query_interface().raycast_closest(origin, unit_direction, max_distance)
```

This is a PhysX scene query returning a dictionary. It is not a method of the
tensor `SimulationView`, and does not establish Newton backend support.

For higher-fidelity sensor simulation (LiDAR scan patterns, multi-ray, vendor sensor models, depth/radar/acoustic), see [sensors.md](isaac-sim-sensor.md), [isaac-sim-sensor.md](isaac-sim-sensor.md), and [isaac-camera.md](isaac-camera.md). They use `isaacsim.sensors.experimental.rtx` and `.physics`. Viewport helpers do not replace those sensors.

---

## Part 7 — Physics-to-USD readback

The most common silent bug: reading authored USD transforms instead of simulated state.

| Source | Returns | When to use |
|---|---|---|
| `UsdGeom.XformCache.GetLocalToWorldTransform()` | Composed USD transform (can lag physics writeback) | Editor-time queries, before play |
| `RigidPrim.get_world_poses()` | Simulated state, batched `(N, 3)` / `(N, 4)` WXYZ | During simulation |
| `Articulation.get_world_poses()` | Simulated state for articulated bodies | Articulated robots |

Select the body explicitly, even for a `(1, 3)` position. USD transforms and an old `UsdGeom.XformCache` are not an independent live-state query.

### Why XformCache is wrong during sim

`updateToUsd=True` writes physics state to **Fabric**, not the USD stage layer. `XformCache` reads the USD layer. Result: it can return initial poses.

### RigidPrim / GeomPrim pattern (Kit 110)

```python
from isaacsim.core.experimental.prims import RigidPrim
import isaacsim.core.experimental.utils.app as app_utils

rp = RigidPrim(paths="/World/Dice/Die_*")
app_utils.play(commit=True)
pos_wp, quat_wp = rp.get_world_poses()  # warp arrays
positions = pos_wp.numpy()  # (N, 3)
quaternions = quat_wp.numpy()  # (N, 4) [w, x, y, z]
```

Experimental `RigidPrim.set_velocities(linear_velocities, angular_velocities)` is the pinned setter (batched). Classic `isaacsim.core.prims.RigidPrim.set_linear_velocities` is a different class. Apply velocities after play/reset, not via ignored USD `physics:velocity` as a runtime write.

### Articulation pattern (Kit 110)

```python
from isaacsim.core.experimental.prims import Articulation
import isaacsim.core.experimental.utils.app as app_utils

robot = Articulation("/World/Robot")
app_utils.play(commit=True)  # required for tensor data
pos_wp, quat_wp = robot.get_world_poses()
dof_positions = robot.get_dof_positions().numpy()  # (N, num_dofs)
dof_velocities = robot.get_dof_velocities().numpy()
J = robot.get_jacobian_matrices().numpy()  # for IK
```

Classic `isaacsim.core.api.articulations.Articulation` / `isaacsim.core.api.prims.RigidPrim` still load; they are superseded by `isaacsim.core.experimental.*`, not removed. `omni.isaac.*` paths were removed.

> **Migration:** [Renaming Extensions](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_4_5/extensions_renaming.html). Experimental Articulation / RigidPrim: [Python scripting index](https://docs.isaacsim.omniverse.nvidia.com/latest/python_scripting/index.html).

### Quaternion convention

USD/Isaac tensor APIs use `[w, x, y, z]`; Isaac Lab 3 and scipy use `[x, y, z, w]`. Convert once at the boundary:

```python
from scipy.spatial.transform import Rotation

r = Rotation.from_quat([quat[1], quat[2], quat[3], quat[0]])
euler = r.as_euler("xyz", degrees=True)
```

---

## Part 8 — Common gotchas

1. A collider without a rigid-body owner is static; a rigid-body ancestor can own child colliders.
2. Compound bodies keep one intended rigid-body owner, not a nested body per collider.
3. **Kinematic bodies**: use `CreateKinematicEnabledAttr().Set(True)`, not enable/disable on RigidBodyAPI.
4. **`Cube.size=1.0`** = half-extent 0.5. Use `size=2.0` if you want scale ops to equal half-extents.
5. **`physics:velocity` USD attributes are ignored** by PhysX at runtime. Use `RigidPrim.set_velocities(...)` **after play/reset**.
6. **`physics:angularVelocity` is in degrees/second** on the USD attribute; runtime tensors often use rad/s. Convert at the boundary.
7. **`World.step(render=True)`** (or the selected framework's step) advances physics with render. `app.update()` does not define the physics timestep.
8. **Experimental sensors need `app_utils.play(commit=True)`** (or the framework play) before `get_data()`; do not call `initialize()` from the legacy `World` flow.
9. **`get_rigid_body_state()` is not the experimental API**; use `RigidPrim.get_world_poses()`. Dynamic Control (`dc.get_rigid_body_pose`) is not a current recommended surface at this pin.
10. **Contact-chain behavior depends on the solver and setup** (see [Worked Example 4: Newton's Cradle](physics-simulation-examples.md)).
11. **Tunneling at high spin rates**: inspect collision geometry, CCD, and timestep; verify the task again after any approximation change.
12. **Before controller diagnosis**: verify a `PhysicsScene`, active stepping owner, and expected collision APIs are present.

---

## Worked examples (impact, vibratory feeder, gyro, cradle, escapement)

See [physics-simulation-examples.md](physics-simulation-examples.md).
