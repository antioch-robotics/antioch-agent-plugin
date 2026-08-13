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

Do not copy that USD by itself. It is a reference-heavy stock asset; relative
mesh/material/payload paths resolve only when the machine's Isaac asset root
and its directory layout remain intact. For a team asset, use
`antioch.load_asset(name, prim_path="/World/Robot", version=...)` instead (the
catalog flow is covered by `antioch-platform`'s assets reference).

Some asset revisions open with no default prim or no descendants, including
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
