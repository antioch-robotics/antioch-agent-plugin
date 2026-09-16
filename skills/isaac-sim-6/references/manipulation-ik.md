# Manipulation IK

Adapted from NVIDIA [`manipulation-ik/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/manipulation-ik/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Control manipulator arms with differential IK, schema-native poser workflows, grasp frames, fixed-joint grasping, and hybrid IK plus joint-space motion.

Patterns reference Isaac Sim docs and upstream example files; embedded code is a pattern sketch, not the canonical source. Always read the linked example; the pinned upstream snapshot is provenance, not proof every API ships at the Isaac Sim 6.0.1 / Kit 110.1.2 runtime pin — confirm symbols with the research index before relying on them.

## Upstream source examples

| Script | Purpose | Arguments |
|---|---|---|
| [`scripts/differential_ik_sketch.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/manipulation-ik/scripts/differential_ik_sketch.py) | Conceptual sketch for custom Jacobian-based differential IK | see script --help |
| [`scripts/robot_poser_example.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/manipulation-ik/scripts/robot_poser_example.py) | Schema-native IK + named-pose workflow using isaacsim.robot.poser | see script --help |

## When to use

- Control an articulated arm to reach, grasp, transport, place.
- Set up IK-based end-effector control (vs joint-space).
- Store reusable robot poses as named poses and apply them later.
- Set up grasping (FixedJoint, `SurfaceGripper`, contact-based).
- Validate manipulation success with a feedback loop.

## Pick the right IK stack

| Stack | Module | When |
|---|---|---|
| Differential IK on `Articulation` | `isaacsim.core.experimental.prims.Articulation` + custom Jacobian solver | specialized direct end-effector control; no maintained example wrapper |
| Schema-native IK + named poses | `isaacsim.robot.poser.RobotPoser` (LM solver via `isaacsim.robot.poser.IKSolverRegistry`) | offline pose authoring, persisted "pick_position" / "approach" poses |
| Obstacle-aware reactive | `isaacsim.robot_motion.cumotion.RmpFlowController` via [motion-generation](motion-generation.md) | dynamic obstacle avoidance, reactive trajectories |
| Pinocchio / PINK | `isaacsim.robot_motion.pink.PinkIKController` | alternative full IK stack with joint limits / task hierarchies |
| Lula motion generation | `isaacsim.robot_motion.lula` + `isaacsim.robot_motion.motion_generation` | legacy; supported but use one of the above for new work ([rename map](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_4_5/extensions_renaming.html)) |

## Upstream example files (canonical source)

Under [`source/standalone_examples/api/isaacsim.robot_motion.examples/manipulation/`](https://github.com/isaac-sim/IsaacSim/tree/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/standalone_examples/api/isaacsim.robot_motion.examples/manipulation) in the pinned upstream source tree (source examples, not local paths):

| Topic | Path |
|---|---|
| Follow target | `follow_target.py` |
| Pick and place | `pick_place.py` |
| Stacking | `stacking.py` |
| Multiple tasks | `multiple_tasks.py` |

Shared implementations are under
`source/extensions/isaacsim.robot_motion.examples/isaacsim/robot_motion/examples/manipulation/`
(`ManipulationScenario`, robot configurations, controllers, and interactive task backends).

Classic manipulators examples live in the `isaacsim.robot.manipulators.examples` extension (shipped under `extsDeprecated`: deprecated, not removed). Enable it explicitly through `SimulationConfig.extensions` when porting classic Franka examples.

> **Migration:** use `isaacsim.robot_motion.examples` for maintained manipulation examples.

## Docs references

| Topic | URL |
|---|---|
| Pick-and-place tutorial | https://docs.isaacsim.omniverse.nvidia.com/latest/robot_setup_tutorials/tutorial_pickplace_example.html |
| Setup a manipulator (import / assemble) | https://docs.isaacsim.omniverse.nvidia.com/latest/robot_setup_tutorials/tutorial_import_assemble_manipulator.html |
| Physics fundamentals (joints, schemas) | https://docs.isaacsim.omniverse.nvidia.com/latest/physics/simulation_fundamentals.html |
| Python scripting index | https://docs.isaacsim.omniverse.nvidia.com/latest/python_scripting/index.html |

## Differential IK pattern (modern `Articulation`)

The conceptual solver does:

1. Get `robot.get_jacobian_matrices()`. Including the batch axis, fixed-base
   shape is `(N, num_links - 1, 6, num_dofs)`; floating-base shape is
   `(N, num_links, 6, num_dofs + 6)`. Only floating bases have six base columns.
   Fixed-base link rows omit the root: use `end_effector_link_index - 1`.
   Resolve the arm's DOF columns by name; do not drop six columns on a fixed base.
2. Compute the 6-DOF pose error from current EE pose to goal.
3. Apply the chosen solver to map error -> joint delta:
   - `damped-least-squares`: `dq = J^T (J J^T + lambda^2 I)^-1 . error` (default).
   - `pseudoinverse`, `transpose`, `singular-value-decomposition` also available.
4. Push as `set_dof_position_targets(current + dq, dof_indices=arm_dofs)`.

`differential_ik_step(arm, end_effector, end_effector_link_index, target_pos, target_quat, arm_dofs, method, damping, scale)` — compute Jacobian, solve IK delta, apply via `set_dof_position_targets`. This is a conceptual pattern; use the maintained RMPflow examples for production control.

See [`scripts/differential_ik_sketch.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/manipulation-ik/scripts/differential_ik_sketch.py).

Tuning (start conservative, increase after stability):

| Parameter | Conservative | Moderate | Aggressive |
|---|---|---|---|
| `damping` | 0.1 | 0.05 | 0.01 |
| `max_delta` per step | 0.02 rad | 0.05 rad | 0.10 rad |
| Drive `stiffness` | 200 | 400 | 800 |

Aggressive settings cause PhysX divergence under payload.

### Hybrid IK + joint-space (arms with < 6 DOF)

Pure differential IK on under-actuated arms fails on:
- Large lateral transport with payload.
- Configurations near kinematic singularities.
- Sweeping through joint limits.

Pattern: IK for precision (approach, descent, final placement), joint-space interpolation for long transport (lift, traverse, descend). See the maintained manipulation controllers for sequenced examples.

## Schema-native IK + Named Poses (RobotPoser)

For pose authoring, persistence, and replay use `isaacsim.robot.poser`. It owns the kinematic chain and IK implementation and stores named-pose data as `IsaacNamedPose` prims on the robot.

`solve_and_store_pose(stage, robot_prim, start_prim, end_prim, target_pos, target_orient, pose_name)` — validate schema, solve IK, apply joints, store as named pose. `apply_stored_pose` / `export_all_poses` for replay and persistence.

See [`scripts/robot_poser_example.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/manipulation-ik/scripts/robot_poser_example.py).

Standalone helpers (no `RobotPoser` needed) for FK / DOF target application:

```python
from isaacsim.robot.poser import apply_joint_state, apply_joint_state_anchored

apply_joint_state(stage, robot_prim, joint_values)  # FK off-sim / DOF targets when playing
apply_joint_state_anchored(
    stage,
    robot_prim,
    joint_values,  # keep anchor at world pose
    anchor_prim=base_link_prim,
)
```

The IK solver is pluggable via `isaacsim.robot.poser.IKSolverRegistry`; the bundled LM solver (`isaacsim.robot.poser.lm_ik`) is the default.

## Obstacle-aware motion controllers

For cuMotion and RMPflow controller setup, including world binding,
`RobotState`, control-loop timing, reset sequencing, supported-robot configs,
and obstacle synchronization, use [motion-generation](motion-generation.md). This skill only decides
when that stack is appropriate and validates the manipulation/grasp side. Use
documented loaders; do not hand-construct controller configs.

## Grasp frame discovery (do this first)

Most assets do not ship with a grasp frame. Before any IK:

1. Inspect the gripper USD; find the frame at the closed-finger center.
2. If absent, add a child Xform of the gripper link positioned at the grasp center; mark it with `IsaacSiteAPI` (`ApplySiteAPI` from `robot_schema`) so downstream tools recognize it.
3. Use that site as the IK target. The goal pose is where the *object center* sits when grasped, not where the gripper body is.

## Grasping

### `FixedJoint` (assisted rigid grasps only)

Pattern source: the maintained interactive `pick_place_task.py` plus `UsdPhysics.FixedJoint`. Always compute the gripper -> object relative transform at the moment of contact; never hardcode the offset. Hardcoded offsets + high stiffness produce PhysX snap and explosion.

Friction-only parallel grasps on a free rigid body are marginal: they may hold on lift but slip under transport acceleration. When the task allows an assisted grasp, prefer `FixedJoint` over tuning grip force or friction: attach at contact, keep the gripper visually closed, and remove the joint on release so the object settles under gravity.

For strict contact-only tasks, the object must move through the gripper's collision/contact forces. Do not use `FixedJoint`, D6 joints, attachments, pose-follow, object pose writes, kinematic holds, disabled dynamics, or post-release stabilization as the success path.

### `SurfaceGripper` (vacuum / magnetic, used by UR10 example)

Pattern source: `manipulation/ur10_palletizing.py` (`DirectSurfaceGripper`).

```python
from isaacsim.robot.surface_gripper import _surface_gripper as surface_gripper

iface = surface_gripper.acquire_surface_gripper_interface()
gripper_path = f"{end_effector_path}/SurfaceGripper"
iface.close_gripper(gripper_path)  # attach
iface.open_gripper(gripper_path)  # release
status = iface.get_gripper_status(gripper_path)  # GripperStatus.{Open,Closed}
```

Authored on the robot via `usd.schema.isaac.robot_schema.CreateSurfaceGripper`.

### Grasp dataset workflow

For generating grasp datasets, see `isaacsim.replicator.grasping` (`GraspingManager`, `GraspPhase`) and `source/standalone_examples/api/isaacsim.replicator.grasping/grasping_workflow_sdg.py`.

## Grasp validation (feedback loop)

Before executing or capturing a manipulation demo, validate the target object. Any
object claimed as picked/pulled/placed/pushed/grasped must be physics-backed: a rigid
body or articulation state, collision geometry, task-appropriate mass/inertia, and
runtime pose readback (physics view or prim API). Visual-only meshes are fine for
probes and debug, but the object in a final claimed result must be physics-backed.

After executing, validate **object state** (not just tool pose) at three gates, in order, stopping at the first failure:

| Gate | Required evidence | Failure action |
|---|---|---|
| Grasp / contact | fingers around object; object within ~2 cm of grasp frame | adjust grasp frame offset or IK target |
| Lift / hold | object leaves the support and holds above the lift threshold for a measured window; XY drift bounded; velocity settles | gripper not engaged; grasp offset wrong (assisted: `FixedJoint` missing/wrong) |
| Place / release | gripper opens, object settles on support within ~5 cm; final pose/velocity meet thresholds; previously placed objects still pass | transport trajectory missed target |

- Success is measured from object pose/orientation, not tool pose: a tool can converge while the object slips, ejects, or stays high. Smooth motion is necessary but not sufficient.
- Placement phase gates should use released object pose, not only end-effector convergence.
- Treat motion-phase timeouts as failures unless the phase is an intentional dwell or settle.
- Multi-object: revalidate the already-placed prefix after every later approach, place, and retreat.
- Require visual evidence from the latest run: fixed-camera video or screenshots showing the robot, object, support surface, grasp/contact area, and markers.
- Strict contact-only tasks must succeed through contact forces alone (no `FixedJoint` or other assistance, see above). If a gate fails, report it with numeric and visual evidence; never add hidden pose assistance to make the output look successful.

## Rules

1. Read the local example first; this skill describes patterns, not syntax.
2. Always create or identify a grasp frame (`IsaacSiteAPI`) before IK.
3. Start conservative with IK gains; increase only after confirming stability.
4. The URDF importer applies `PhysxArticulationAPI` automatically; if you author articulations manually, apply it on the base link.
5. Run scripts in the Antioch simulator service; a plain script calls `antioch.start_simulation()` before simulator imports, and a managed scenario declares `SimulationConfig` instead of constructing `SimulationApp`.
6. Jacobian layout depends on base type: only a floating base has six leading base columns; fixed-base rows omit the root link.
7. Store reusable poses with `store_named_pose`; do not re-solve IK from scratch every session.
8. `print()` is unreliable in headless mode; use file writes for debug logging.
9. Validate visually at every phase. Smooth motion is not successful manipulation.
10. Hybrid IK + joint-space is the pragmatic default for arms with < 6 DOF.

## Lessons

- SO-101 5-DOF: pure DLS IK converges for local moves (~0.008 m error) but diverges on lateral transport under load. Hybrid is required.
- `FixedJoint` with hardcoded offset + high stiffness causes PhysX snap and explosion. Compute the offset at grasp time.
- Wrong Jacobian row or DOF selection breaks IK silently. Verify base type and named link/DOF indices before solving.
