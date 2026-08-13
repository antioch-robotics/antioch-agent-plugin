# IK stacks — Isaac Sim 6.0.1 / Kit 110.1.2

Detail reference for the manipulation domain of the `isaac-sim-6` skill. All imports stay
inside function bodies (lazy-import invariant). Differential IK on the
experimental `Articulation` is covered in `references/manipulation.md`; this
file covers the other three stacks and the shared details.

## Jacobian layout (applies to every differential-IK user)

`Articulation.get_jacobian_matrices()` returns one Jacobian per link; the
shape depends on the base:

```
fixed base:    (num_links - 1, 6, num_dofs)     — NO virtual base block
floating base: (num_links,     6, num_dofs + 6) — 6 free-root columns appended at the END
rows    = one per link; the EE row is at end_effector_link_index - 1
```

On a fixed-base arm the only slice that matters is dropping the trailing
gripper DOF columns. Forgetting it is silent: IK still returns values, they
are just wrong. Always:

```python
jacobian = arm.get_jacobian_matrices().numpy()
j_ee = jacobian[:, arm.end_effector_link_index - 1, :, :arm_dofs]
```

`arm_dofs` is the count of actuated arm joints (exclude gripper fingers unless
you intend IK to move them).

### DLS method config

| Key | Meaning | Default start |
|---|---|---|
| `scale` | step scaling | 1.0 |
| `damping` | DLS damping λ; higher = stabler, slower | 0.05 (0.1 conservative) |
| `min_singular_value` | singularity guard | 1e-5 |

Other methods: `pseudoinverse`, `transpose`, `singular-value-decomposition`.
Stay on `damped-least-squares` unless you have a measured reason to switch.

## RobotPoser — schema-native IK + named poses

`isaacsim.robot.poser` wraps a kinematic chain declared by
`usd.schema.isaac.robot_schema` (`IsaacRobotAPI`, applied by the URDF/MJCF
importer) and provides IK plus a named-pose library persisted as
`IsaacNamedPose` prims on the robot. The solver is pluggable via
`IKSolverRegistry` (import it from `isaacsim.robot.poser`); the bundled LM
solver (`usd.schema.isaac.robot_schema.lm_ik`) is the default.

```python
def author_pick_pose(stage, robot_prim, start_prim, end_prim) -> None:
    from isaacsim.robot.poser import RobotPoser, Transform, store_named_pose, validate_robot_schema

    if not validate_robot_schema(robot_prim):  # IsaacRobotAPI present?
        raise ValueError(f"{robot_prim.GetPath()} has no IsaacRobotAPI — re-import via URDF/MJCF")
    poser = RobotPoser(stage, robot_prim, start_prim, end_prim)
    target = Transform(t=[0.5, 0.2, 0.8], q=[1, 0, 0, 0])
    result = poser.solve_ik(target)
    if result.success:
        poser.apply_pose(result.joints)
        store_named_pose(stage, robot_prim, "pick_position", result)


def replay_pick_pose(stage, robot_prim) -> None:
    from isaacsim.robot.poser import apply_pose_by_name, export_poses

    apply_pose_by_name(stage, robot_prim, "pick_position")
    export_poses(stage, robot_prim, "/tmp/poses.json")
```

Also exported: `list_named_poses`, `delete_named_pose`, `import_poses`.

Standalone helpers (no `RobotPoser` needed) for FK / DOF targets:

```python
from isaacsim.robot.poser import apply_joint_state, apply_joint_state_anchored
```

- `apply_joint_state(stage, robot_prim, joint_values)` — FK off-sim, DOF
  targets when playing.
- `apply_joint_state_anchored(..., anchor_prim=base_link_prim)` — keeps the
  anchor link at its world pose while the chain moves.

Use RobotPoser for poses authored once and replayed (approach, home,
pick_position) — it removes per-run IK solve variance from the acceptance
loop. Live reactive control still belongs to differential IK.

## cuMotion RMP — obstacle-aware reactive motion

Module: `isaacsim.robot_motion.cumotion`. Use the documented loaders; do not
hand-construct configs.

```python
def make_rmp_controller(arm):
    from isaacsim.robot_motion.cumotion import CumotionWorldInterface, RmpFlowController, load_cumotion_supported_robot
    from isaacsim.robot_motion.experimental.motion_generation import RobotState

    robot = load_cumotion_supported_robot("franka")  # bundled config + chain
    world = CumotionWorldInterface(...)  # populate with obstacle prims
    ctrl = RmpFlowController(
        cumotion_robot=robot, cumotion_world_interface=world, robot_joint_space=arm.dof_names, robot_site_space=robot.robot_description.tool_frame_names()
    )
    ctrl.reset(estimated_state, None, t=0.0)  # REQUIRED before the first forward()
    return ctrl, RobotState
```

Per step: `action = ctrl.forward(estimated_state=..., setpoint_state=...,
t=sim_time)` — `t` (the sim clock) is a required argument, and the setpoint
carries the target EE pose. `reset()` must be called before
the first `forward()` of each episode — forgetting it yields stale-state
plans that look like random motion.

Reach for RMP when the scene has obstacles the arm must react to; for an open
tabletop, differential IK is simpler and easier to validate.

## PINK — Pinocchio-based IK

Module: `isaacsim.robot_motion.pink`. Prefer it over the legacy Lula-era stack
at `isaacsim.robot_motion.motion_generation` for new code.

```python
def make_pink_controller(arm, tool_frame: str):
    from isaacsim.robot_motion.pink import PinkIKController, load_pink_supported_robot

    robot = load_pink_supported_robot("franka")  # bundled PinkRobot wrapper
    return PinkIKController(
        pink_robot=robot,
        robot_joint_space=arm.dof_names,
        robot_site_space=[tool_frame],
        tool_frame=tool_frame,
        dt=1.0 / 120.0,  # required — QP integration timestep
    )
```

- The built-in frame task tracks the tool-frame pose with `position_cost` /
  `orientation_cost` weights, plus posture regularization via `posture_cost`;
  extra PINK tasks (joint, relative-frame) go in through `extra_tasks` — this
  is the draw over plain DLS: hard joint limits and a task hierarchy instead
  of one least-squares solve.
- For a non-bundled robot, use `load_pink_robot(...)` with your own
  configuration rather than `load_pink_supported_robot`.

Choose PINK when you need joint limits or null-space posture behavior that DLS
cannot express; choose cuMotion when obstacle avoidance is the requirement.

## Migration notes (6.0)

All `omni.isaac.*` motion/manipulator modules are removed (SKILL.md namespace
map); their 6.0.1 homes are `isaacsim.robot_motion.motion_generation`,
`isaacsim.robot.experimental.manipulators`, and
`isaacsim.robot.surface_gripper` — per-symbol mappings: `research_search`
the old name.

The `isaacsim.robot_motion.lula` module exposes no solver API: the Lula-era APIs
(`LulaKinematicsSolver`, `RmpFlow`, …) live on at
`isaacsim.robot_motion.motion_generation` as the legacy stack — prefer PINK or
cuMotion for new code.

If an API's shape is uncertain, probe it on the machine with `antioch run`
(see `antioch-platform`) instead of emitting it from memory — 4.5-era advice is
actively wrong against 6.0.
