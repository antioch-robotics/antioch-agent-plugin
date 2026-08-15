# Manipulation — Isaac Sim 6.0.1 on Antioch

Detail reference for the `isaac-sim-6` skill.

Code written here is authored locally (no simulator installed) and executed
remotely on Antioch GPU machines running Isaac Sim 6.0.1 / Kit 110.1.2.
Simulation commands can stream the remote GUI into the browser, but grasping
and placement still need numeric checks against simulated state. A visual
inspection helps debug a run; it is not the pass criterion.

- Rule #1 (lazy imports), the 6.0 namespace map, boot/step/readback ordering,
  and the XformCache trap: load this skill's `SKILL.md` first. Every
  snippet here assumes those invariants — `pxr`/`omni`/`isaacsim` imports live
  inside function bodies only.
- Dispatch, logs, and metrics/artifact readback (`antioch scenario
  show/logs/download`): the `antioch-platform` skill owns that. This skill is only
  about writing correct manipulation code.
- Contact geometry, instanced colliders, misleading metrics, and deformable
  reset ordering: load `references/manipulation-physics.md` before changing a
  controller or grasp threshold.

## Acquire and inspect a stock robot

The pinned 6.0.1 asset tree uses the asset-root-relative Franka path
`/Isaac/Robots/FrankaRobotics/FrankaPanda/franka.usd`; the shorter
`/Isaac/Robots/Franka/franka.usd` is not the canonical path shown by the
harvested examples. Acquire it on the machine and reference it in place; the
old `isaacsim.robot.manipulators.examples` and `isaacsim.core.utils.nucleus`
acquisition shortcuts do not exist in this image — importing either raises
`ModuleNotFoundError`. `get_assets_root_path()` plus the stage reference
helper is the portable path:

```python
def add_franka(stage) -> None:
    import isaacsim.core.experimental.utils.stage as stage_utils
    from isaacsim.storage.native import get_assets_root_path

    stage_utils.add_reference_to_stage(usd_path=get_assets_root_path() + "/Isaac/Robots/FrankaRobotics/FrankaPanda/franka.usd", path="/World/Franka")
```

The verified 6.0.1 stack from there: wrap the referenced Franka in
`SingleManipulator` with a `ParallelGripper` (`isaacsim.robot.manipulators`)
and drive it with the Franka RMPflow motion-policy config — run live, that
combination produced real finite pick-and-place trajectories.

### Newton and the shipped Franka experimental IK

The shipped Franka experimental IK helper is a PhysX path on Isaac Sim
6.0.1. With Newton active, the helper's target translation and Jacobian path
does not agree with the composed Franka articulation's DOF metadata: the
Newton tensor reports an extra DOF and the later Jacobian has the wrong shape.
This is an upstream integration defect, not a reason to slice tensors until
the numbers fit. Use the default PhysX boot for this helper:

```python
import antioch

antioch.boot(physics_engine="physx")
```

If Newton is required, use a Newton-native controller with an explicit,
verified DOF map and calibrate its joint limits and end-effector frame against
the actual composed asset. Low-level joint targets are not an equivalent IK
fix. Record the engine, `dof_names`, Jacobian shape, and solver result in the
run so a future Isaac update can prove the boundary is gone.

Do not copy that USD by itself. It is a reference-heavy stock asset; relative
mesh/material/payload paths resolve only when the machine's Isaac asset root
and its directory layout remain intact. For a team asset, use
`antioch.load_asset(name, prim_path="/World/Robot", version=...)` instead (the
catalog flow is covered by `antioch-platform`'s assets reference).

Some asset versions open with no default prim or no descendants, including
reports against a Franka file. Prove what the composed file contains before
building IK around a path:

```python
def inspect_robot_file(usd_path: str) -> None:
    from pathlib import Path
    from pxr import Usd

    stage = Usd.Stage.Open(usd_path)  # composed stage, not Sdf.Layer.Open
    default = stage.GetDefaultPrim()
    print("defaultPrim:", default.GetPath() if default else None)
    prims = [prim.GetPath() for prim in stage.Traverse()]
    print("prim count:", len(prims), "first paths:", prims[:20])
    variants = sorted(Path(usd_path).parent.rglob("*_instanceable.usd"))
    print("instanceable candidates:", [str(path) for path in variants])
```

If the count is zero, check the asset root, payload loading, and any
`*_instanceable.usd` variant before guessing a prim path. A USD that merely
opens successfully is not evidence that its articulation is composed.

## SO-101 pick and place — the composition that passes

Start here. A closed-loop pick and place is proven on Isaac Lab 3.0 and ships
with the repository as `so101_pick_place`, in the `isaac-lab` example project:

```bash
antioch suite run so101-pick-place                     # the nominal case
antioch suite run so101-pick-place-envelope            # 18 conditions
```

It pairs the `robots/so101-follower-jawfix@v1` arm with a **native 30 mm,
20 g cuboid on a ground plane**. It deliberately does not use the 40 mm
catalog cube or the workcell — that pairing is the next section, and it has
never passed a live lift.

The controller is a feedback policy, not a replayed joint trajectory: it reads
the object **position** every step, corrects the task-space goal during lift
and transport, and uses two measured joint seeds to select reachable kinematic
branches. It does not read object orientation — the end-effector orientation is
fixed from the initial radial angle — so this is position feedback, not pose
feedback. The phase schedule is fixed; nothing here is learned.

Measured envelope, suite run `ad085a15c25e4ffdbc15c67eea2370ac`: 12 of 18
members passed, five failed, and the nominal member was cancelled by a
machine dispatch failure. A direct retry of nominal
(`e543ba67127843f291fa766b0a5866cd`) passed, so the 18-case scenario verdict
count is 13 passed and five failed, in 11m 32s across two machines.

**All 18 cases ended with the object inside the bin.** The five that did not
pass every check tripped the gates below, and the recorded values say exactly
which:

| Case | Sustained lift | Displacement | In bin | Checks it failed | Measured / limit |
|---|---:|---:|:--:|---|---:|
| `pose-left` | 0.059792 m | 0.262 m | yes | grasp gate | 0.020146 / 0.020 m |
| `yaw-45` | 0.051867 m | 0.205 m | yes | grasp gate, containment | 0.041784 / 0.020 m, 0.290618 / 0.25 m |
| `scale-40mm` | 0.040154 m | 0.219 m | yes | sustained lift, grasp gate, containment | 0.040154 / 0.045 m, 0.042234 / 0.020 m, 0.281076 / 0.25 m |
| `friction-0p6` | 0.071977 m | 0.237 m | yes | grasp gate | 0.020476 / 0.020 m |
| `friction-2p0` | 0.052157 m | 0.230 m | yes | grasp gate, place tolerance | 0.025079 / 0.020 m, 0.035132 / 0.035 m |

The corrected values are the measurements that decide these gates. The grasp
value is the maximum closure-offset error sampled on every lift and traverse
step, and the lift value is the minimum height sustained through traverse.
`scale-40mm` therefore fails the sustained-lift gate at 0.040154 m, while
`yaw-45` reaches 0.290618 m and fails the unchanged containment maximum-height
limit. `friction-2p0` still misses the place tolerance by 132 micrometres and
also records a 0.025079 m maximum closure-offset error.

Copy this scenario before composing your own. Eight checks with measured
tolerances decide the outcome, so a miss is recorded as a failure rather than
described as a success.

## SO-101 catalog composition — pick not proven

The following is the smallest SO-101 scene that a newcomer should compose for a
pick investigation. It uses the tenant catalog, not guessed Isaac asset-root
paths. Pin every version: a moving robot version can change jaw collision
geometry without changing the Python controller. The scene settles, but the
catalog composition has no passing live pick certificate yet.

| Role | Catalog asset and version | Place it at | Important detail |
|---|---|---|---|
| Arm | `robots/so101-follower-jawfix@v1` | `/World/Robot` | Jaw collision uses convex decomposition. Do not start with `robots/so101-follower@v2`; its convex-hull jaw makes a roughly 21-degree wedge that clamps or throws objects. |
| Surface | `hackathon/so101-workcell@1.0.0` | `/World/Workcell`, translate `(0, 0, 0.384) m` | Static box tabletop. The corrected top is `z=0.420 m`; its footprint is `x=+-0.300 m`, `y=+-0.250 m`. |
| Object | `hackathon/so101-cube-set@1.0.0` | `/World/CubeSet` | Keep only `Cube_Medium` for the first run. It is a 40 mm, 0.0384 kg dynamic box whose local center is `(0, 0, 0.020)`. Disable the small and large distractors before reset. |

The workcell asset has an authored `RobotMount` hint of `(0, -0.180,
0.420) m`. Use that as the robot root pose. The workcell root translation is
not optional: the USDA's `scale`-then-`translate` xform order scales its
0.400 m tabletop translation by the 0.040 m Z scale, so the unshifted tabletop
is only `z=-0.004..0.036 m` in the composed stage. Translate the workcell root
by `(0, 0, 0.384) m` and assert its simulated tabletop bounds are
`z=0.380..0.420 m`. Put the cube-set parent at
`(0.220, -0.180, 0.420) m`; the medium cube then rests on the tabletop with
its bottom at `z=0.420 m` and has a 0.220 m reach radius from the arm base.
Add a shallow, open bin as scene-native box geometry at `(0.115, 0.010,
0.420) m` (100 mm inside width, 25 mm walls, 6 mm floor). These coordinates
are the measured ground-plane demo translated to the workcell's arm mount. Do
not scale any of the catalog assets. Measure the composed bboxes and assert the
tabletop and cube bottom before `World.reset()`.

This composition is deliberately explicit about the cube-set transform: the
asset contains three dynamic cubes, and loading the whole set without removing
the other two creates extra contacts and makes a first run non-diagnostic. The
catalog layout above is the scene recipe. Do not describe it as pick-ready until
the live lift gate below passes on this exact composition.

### Live catalog dispatch — pick failed

The exact catalog assets and transforms above were dispatched on Isaac Sim
6.0.1 on 2026-08-14. Run
`f3d9d43ece8d4530a61efdb8f6a06629` in project `SO-101 Pick Proof` loaded and
settled the scene, then failed the `cube_lifted` check:

| Evidence | Measured value |
|---|---:|
| Time from dispatch start to the settled-cube sample | `13.599 s` |
| Complete `antioch run` duration | `18.332 s` |
| Settled medium-cube pose | `(0.2199999, -0.1800002, 0.4399999) m` |
| Cube pose at grasp phase | `(0.2459001, -0.1784823, 0.4400000) m` |
| Cube pose at lift phase | `(0.2523814, -0.1780725, 0.4400000) m` |
| Measured lift | `0.00000006 m` |
| Measured slip | `0.006494 m` |

The arm reached the recorded approach, descend, close, and lift positions, but
the jaw pushed the cube in +X instead of lifting it. The cube speed was
`0.000 m/s` at readback because it remained on the tabletop. The check failed;
the object did not leave the surface.

The composition settled on the first clean catalog inspection. There were 20
`so101_pick` controller dispatches in this investigation: 18 failed numeric
checks and 2 errored while correcting the probe and API usage. None lifted the
cube. The ground-plane certificates below are historical evidence only; they do
not certify this catalog workcell.

The live articulation exposed the DOFs
`shoulder_pan, shoulder_lift, elbow_flex, wrist_flex, wrist_roll, gripper` and
the frame at
`/World/Robot/Geometry/base_link/shoulder_link/upper_arm_link/lower_arm_link/wrist_link/gripper_link/gripper_frame_link`.
`Articulation` has no public `prim_paths` attribute, and
`RigidPrim.get_velocities()` returns a `(linear, angular)` tuple. These API
details corrected the probe; they did not change the failed physical result.

### Reach and approach numbers

SO-101 has five positioning DOF. Use `gripper_frame_link` as the end-effector
frame. The numbers below came from a ground-plane controller and are not a
validated catalog target: the live catalog controller reached the position
targets but lost its top-down orientation and pushed the cube. Enforcing that
orientation drove the catalog arm into its joint limits. Recalibrate the pose
solver against the composed USD before calling this scene a pick.

| Radial distance from arm base | Safe TCP height above tabletop | Measured tracking error |
|---|---:|---:|
| `0.20 m` | `0.055--0.075 m` | `0.9--1.0 mm` |
| `0.24 m` | `0.055--0.065 m` | `1.5 mm` |
| `0.24 m` | `0.075 m` | `about 4 mm` near the wrist limit |

Use `r <= 0.240 m` and `TCP z <= surface_z + 0.075 m` as the first-run
rule. The fixed fingertip reaches the tabletop at about
`surface_z + 0.006 m`; do not command below it. For the layout above, use
`surface_z + 0.070 m` for approach, lift, and traverse and
`surface_z + 0.010 m` for the grasp. The object center is not the TCP: place
the TCP 28 mm *inboard* of the object along the radial direction
(`GRASP_RADIAL_OFFSET = 0.028 m`). Re-measure the object-to-TCP offset after
the lift and again after the traverse before choosing the release target.

### Gripper and physics settings that settle

- Pre-open the jaw to `q=0.60 rad` (about 54 mm). Above `q ~= 0.95`, the moving
  jaw clears the object band and a close starts with a downward slam instead of
  a lateral sweep.
- Close a flat cube to `q=0.10 rad`. Use `q=0.25 rad` for a round object.
  Do not use the URDF's `10 N*m` effort limit: it produced about 200 N and
  launched a 20 g cube. Cap the gripper at `0.30 N*m`, with stiffness `4` and
  damping `0.3`. Start the arm at stiffness `220` and damping `22`.
- Keep PhysX as the backend. Use a 120 Hz step, TGS, 32 position iterations,
  4 velocity iterations, zero rest offset, and a 0.002 m contact offset. Give
  the jaw and object a friction material with static friction about `1.4`.
  Apply jaw material and collision edits to de-instanced jaw subtrees before
  reset; instance proxies are read-only. The full de-instancing and material
  procedure is in `manipulation-grasping.md`.
- Let each phase settle before starting the next. Run exactly
  `approach -> descend -> close -> lift -> traverse -> lower -> release ->
  let_go -> retreat`. Do not open while lowering or retreat while opening:
  that drops a moving object on the bin rim.

### Numeric pick gate

Log simulated object and end-effector poses, not authored `XformCache` values.
The pick is real only when all of these checks pass:

1. During grasp, object center to grasp site is at most `0.020 m`.
2. During lift, object height rises by at least 80% of the commanded lift and
   object-to-site slip stays below `0.020 m`.
3. After release and at least one second of settling, object speed is below
   `0.010 m/s`, final XY error from the bin center is at most `0.055 m`, and
   the object center is between the bin floor and rim.

The lift check rejects the common false positive where the jaw merely pushes
the cube. A 30--40 mm cube that stays on the tabletop cannot reach the lift
threshold. Save `grasp_offset_m`, `lift_delta_z_m`, `lift_slip_m`,
`place_error_m`, and `residual_speed_mps` as run metrics.

### Failure symptoms

| Symptom | Likely cause | First correction |
|---|---|---|
| Jaw hits the top of the cube | Pre-open `q > 0.95` | Start at `q=0.60`; close laterally. |
| Object wedges, clamps, or shoots sideways | `robots/so101-follower@v2` convex-hull jaw | Use `robots/so101-follower-jawfix@v1`. |
| Cube crosses the table or vanishes on close | `10 N*m` jaw effort or aggressive gains | Cap effort at `0.30 N*m`; use the gains above. |
| IK error grows during transport | Radius above `0.240 m`, high TCP, or pure IK | Stay top-down; use joint-space interpolation for lift/traverse. |
| Arm reaches but misses the object | TCP treated as object center | Subtract the 28 mm radial offset and verify the grasp frame. |
| Cube falls through or the table is near `z=0.036 m` | Workcell `scale` then `translate` order scales the authored 0.400 m Z translation | Translate `/World/Workcell` by `(0, 0, 0.384) m`; assert tabletop `z=0.380..0.420 m`. |
| Object is in the bin footprint but on its rim | Hard-coded or stale held offset | Measure the offset after lift and after traverse. |
| Object appears frozen at its spawn pose | Authored-layer readback | Read `RigidPrim.get_world_poses()` after stepping. |
| Position-only IK reaches the lift point but the cube moves sideways | The catalog arm loses the required top-down orientation | Solve a joint-limited full pose against the live `gripper_frame_link`; the documented position-only recipe is not a pick proof. |
| Full-pose IK drives shoulder or elbow to a limit | The documented top-down target is not reachable for this catalog USD at the stated mount | Stop and recalibrate the TCP pose, mount, and joint limits before changing the jaw effort. |

### Existing run certificate

The controller and geometry were run successfully on Isaac Lab 3.0 / Isaac Sim
6.0.1 with a ground plane at `z=0`, a 30 mm dynamic cube, and the same measured
SO-101 jaw and top-down phases. These are historical ground-plane certificates;
the catalog workcell dispatch above is the current evidence and failed:

| Case | Run ID | Result | Wall time |
|---|---|---|---:|
| centre | `3c09d9cece45417d8c52f4e4d9c5beb7` | 75.2 mm max cube height; 4.3 mm bin error | 3m 10.284s cold |
| offset-left | `afd9c07805d04ee083b1be5862715625` | 71.0 mm max height; 9.2 mm bin error | 21.550s warm |
| offset-right | `bae1100df77843169b41e46051afdf94` | 70.4 mm max height; 6.7 mm bin error | 39.7s warm |

The cold run spent 174 s before `sim.reset()` returned (asset download and
first shader compilation); the warm run's setup was 1.51 s and motion was
10.56 s for 13.42 s of simulated time. Reproduce with `--no-stream`; a
livestream is not part of the manipulation gate.

An independent core Isaac Sim 6.0.1 probe of the catalog assets confirmed the
corrected composition before this section was written: the tabletop read
`z=0.380000..0.419999 m`, and after 120 physics steps the medium cube read
`(0.2200000, -0.1800001, 0.43999997) m`. The live dispatch above then showed
that settling is not the same as grasping. Treat the top-down pose and jaw
approach as hypotheses that require the numeric lift gate, not as a guarantee.

The checked-in example tree is currently held by another writer. When it is
available, add `examples/scenarios/so101_pick.py` under the
`packages/antioch-sim` package. Compose the three catalog assets and bin
exactly as above. Keep the skill section as the placement and failure-mode
reference; the example should only encode the controller and numeric gate,
not a second set of geometry constants.

## Pick the IK stack

| Stack | Module | Use when |
|---|---|---|
| Differential IK on `Articulation` | `isaacsim.core.experimental.prims.Articulation` + per-robot wrapper | Direct end-effector control in a loop; the default for reach/follow/pick |
| Schema-native IK + named poses | `isaacsim.robot.poser.RobotPoser` (LM solver via `usd.schema.isaac.robot_schema.IKSolverRegistry`) | Offline pose authoring; persisted `pick_position` / `approach` poses replayed across runs |
| cuMotion RMP | `isaacsim.robot_motion.cumotion.RmpFlowController` | Obstacle-aware, reactive trajectories in cluttered scenes |
| PINK (Pinocchio) | `isaacsim.robot_motion.pink.PinkIKController` | Full IK with joint limits and task hierarchies |
| Lula-era stack | `isaacsim.robot_motion.motion_generation` | Legacy interfaces (`LulaKinematicsSolver`, `RmpFlow`) — still shipped, but prefer PINK or cuMotion for new code. The `isaacsim.robot_motion.lula` extension module exists but exposes no solver API |

Detail and snippets for RobotPoser, cuMotion, and PINK: `references/manipulation-ik-stacks.md`.

## Grasp frame first

Most gripper assets ship without a grasp frame. Before any IK, establish the
frame at the closed-finger center — authoring an `IsaacSiteAPI` site (below) is
the recommended way because downstream tools recognize it, but computing the
frame in code or reusing an importer-detected site is equally valid. The IK
target is where the *object center* sits when grasped, not where the gripper
body is — a missing or wrong site is the most common cause of "IK converges
but grasps miss".

```python
def ensure_grasp_site(stage, gripper_link_path: str, offset=(0.0, 0.0, 0.103)):
    from pxr import Gf, UsdGeom
    from usd.schema.isaac.robot_schema import ApplySiteAPI

    site_path = f"{gripper_link_path}/grasp_site"
    site = UsdGeom.Xform.Define(stage, site_path)
    site.AddTranslateOp().Set(Gf.Vec3d(*offset))
    ApplySiteAPI(site.GetPrim())
    return site_path
```

Measure the offset from the gripper's closed-finger geometry once and keep it
as a named constant; do not re-derive it per run.

## Differential IK on the experimental Articulation

The pattern: Jacobian → 6-DOF pose error → damped-least-squares solve → joint
delta → position targets.

```python
def ik_step(arm, target_pos, target_quat, arm_dofs: int) -> None:
    ee = arm.end_effector_link
    # Fixed-base Jacobians are (num_links - 1, 6, num_dofs) — no virtual base
    # columns — but gripper DOFs trail the arm DOFs, so ALWAYS slice them off
    # (silent-wrong-IK bug #1). Floating bases append 6 free-root columns.
    jacobian = arm.get_jacobian_matrices().numpy()
    j_ee = jacobian[:, arm.end_effector_link_index - 1, :, :arm_dofs]
    cur_pos, cur_quat = ee.get_world_poses()
    cur_dofs = arm.get_dof_positions().numpy()
    dq = arm.differential_inverse_kinematics(
        jacobian_end_effector=j_ee,
        current_position=cur_pos.numpy(),
        current_orientation=cur_quat.numpy(),
        goal_position=target_pos,
        goal_orientation=target_quat,
        method="damped-least-squares",
        method_cfg={"scale": 1.0, "damping": 0.05, "min_singular_value": 1e-5},
    )
    arm.set_dof_position_targets(cur_dofs[:, :arm_dofs] + dq, dof_indices=list(range(arm_dofs)))
```

`pseudoinverse`, `transpose`, and `singular-value-decomposition` methods also
exist; damped-least-squares is the default and the right starting point.

### Gains — start conservative, increase only after stability

The starting-gains table (DLS damping, max joint delta per step, drive
stiffness/damping) lives in `references/manipulation-grasping.md` — one copy,
kept with the transport-tuning context where it is applied.

Aggressive settings cause PhysX divergence under payload. Symptom: arm jitters
or the sim "explodes" once an object is grasped → drop a row and re-run.

## Arms under 6 DOF — hybrid IK + joint-space

Pure differential IK on under-actuated arms (e.g. 5-DOF SO-101) converges for
local moves (~0.008 m error) but diverges on lateral transport under load,
near singularities, and sweeping joint limits. The pragmatic default:

- **IK** for precision phases: approach, descent, final placement.
- **Joint-space interpolation** for long transport: lift, traverse, descend.

## Grasping

| Mechanism | When | Notes |
|---|---|---|
| `UsdPhysics.FixedJoint` | Rigid-body grasp; simplest reliable pick | Compute the gripper→object relative transform at the moment of contact — never hardcode it |
| `SurfaceGripper` | Vacuum/magnetic end effectors | Attach/release via the surface-gripper interface; authored with `robot_schema.CreateSurfaceGripper` |
| Friction only (finger drives) | Deformables, force-sensitive grasps | Needs tuned drive gains + contact materials; validate via contact forces |

`FixedJoint` with a hardcoded offset plus high stiffness produces PhysX snap
and explosion. Full snippets for both mechanisms and release semantics:
`references/manipulation-grasping.md`.

## Validation — the three-checkpoint loop

Smooth motion is necessary but not sufficient. A pick-and-place run is only
successful when all three checkpoints pass as numeric assertions on simulated
state, each emitted as a run metric so the pass/fail evidence lives in the
scenario's saved results (`run.add_result`, read back with `antioch scenario show
SCENARIO_RUN_ID --json`):

| Checkpoint | Numeric check | Metric | Failure means |
|---|---|---|---|
| Grasp contact | object center within ~2 cm of grasp site | `grasp_offset_m` | wrong grasp-frame offset or IK target |
| Lift success | object Z rises with gripper (Δz ≥ 80% of commanded lift); object–site distance stays < 2 cm through lift | `lift_delta_z_m`, `lift_slip_m` | FixedJoint missing/wrong offset; gripper not engaged |
| Place success | after release + ≥1 s settle, object velocity < 1 cm/s and within ~5 cm of target | `place_error_m`, `residual_speed_mps` | transport trajectory missed; released early |

Read poses from the simulated state (`RigidPrim.get_world_poses()` /
`Articulation.get_world_poses()`), never `XformCache` — see the XformCache
trap in `isaac-sim-6`. A full implementation of the loop:
`references/manipulation-grasping.md`.

Never loosen a threshold to make a run pass — a failing checkpoint is a
manipulation bug, not a metric bug.

## Gotchas (symptom → cause → fix)

1. IK runs without error but the arm moves wrong → Jacobian still includes the
   trailing gripper DOF columns → slice `[..., :arm_dofs]` to drop them (and
   index `end_effector_link_index - 1`).
2. `import isaacsim.robot_motion.lula` gives you no solver API → the
   Lula-era APIs live at `isaacsim.robot_motion.motion_generation` → prefer
   PINK (`isaacsim.robot_motion.pink`) or cuMotion for new code.
3. Any `omni.isaac.*` import → removed shims → use the namespace map in
   this skill's `SKILL.md`.
4. Sim explodes the instant an object is grasped → FixedJoint offset hardcoded
   and/or gains aggressive → compute the offset at contact time; drop a row in
   the gains table.
5. Object reads at its spawn pose forever → reading `XformCache` (authored
   layer) instead of simulated state → `get_world_poses()`.
6. 5-DOF arm diverges mid-transport → pure IK on an under-actuated arm →
   hybrid IK + joint-space for transport phases.
7. Re-solving the same IK every run → poses not persisted →
   `store_named_pose` / `apply_pose_by_name` with RobotPoser.
8. Gripper "closed" but object left behind → SurfaceGripper never engaged or
   status unchecked → assert `get_gripper_status(...) == Closed` before lift
   and emit it as a metric.
9. Module-scope `from pxr import ...` in the scenario file → breaks Antioch
   local discovery → imports inside function bodies (Rule #1).

## Rules

1. Lazy imports always — Rule #1 from `isaac-sim-6` applies to every snippet here.
2. Create or identify a grasp frame (`IsaacSiteAPI`) before any IK.
3. Start conservative with IK gains; increase only after confirming stability.
4. Slice the trailing gripper DOF columns off every fixed-base Jacobian.
5. Never emit `isaacsim.robot_motion.lula` (it exposes no solver API — Lula-era
   APIs are at `isaacsim.robot_motion.motion_generation`) or `omni.isaac.*`
   (removed in 6.0).
6. Compute the FixedJoint offset at grasp time; never hardcode it.
7. Store reusable poses with `store_named_pose`; do not re-solve IK every run.
8. Hybrid IK + joint-space is the default for arms with < 6 DOF.
9. Validate all three checkpoints numerically and emit them as run metrics.
   Smooth motion is not successful manipulation.
10. `print()` is fine for logs, but pass/fail evidence belongs in metrics —
    logs scroll away; the saved run results are the audit surface.

## References (load one level deep when needed)

- `references/manipulation-ik-stacks.md` — RobotPoser named-pose library, cuMotion RMP,
  and PINK usage; Jacobian layout detail; IK method options. Load when setting
  up RobotPoser, obstacle-aware motion, or a non-bundled robot on PINK.
- `references/manipulation-grasping.md` — FixedJoint and SurfaceGripper
  attach/release snippets, drive-gain tuning, and the full three-checkpoint
  validation loop emitting run metrics. Load when implementing grasping or a
  pick-and-place acceptance check.
