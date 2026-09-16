# Motion Generation

Adapted from NVIDIA [`motion-generation/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Build obstacle-aware arm and mobile-base motion with the motion_generation controller substrate; cuMotion with RMPflow is the reference arm implementation.

Owns the generic `isaacsim.robot_motion.experimental.motion_generation` (`mg`)
substrate and the phase-machine workflow for obstacle-aware end-effector
motion. The motion-generation controller is pluggable: cuMotion
(`isaacsim.robot_motion.cumotion`) is the reference implementation here. PINK and
Lula are also motion-generation controllers but are not yet documented in this
skill; see the stack-selection table in [manipulation-ik](manipulation-ik.md) to choose.

Shared substrate lives in sibling skills:

| Need | Read |
|---|---|
| Live session inspection, asset-root checks, screenshots, markers, run videos, final object-pose oracle | [agentic-simulation](../../agentic-simulation/SKILL.md) |
| Grasp frames, contact-only gates, physical grasp validation, IK-stack selection | [manipulation-ik](manipulation-ik.md) |
| Mobile-base planning, footprints, differential-drive/Ackermann kinematics, chase cameras | [navigation-primitives](navigation-primitives.md) |
| Dynamic object collision, rigid bodies, materials, physics readback | [physics-simulation](physics-simulation.md) |
| Headless rendering and video capture | [isaac-sim-rendering](isaac-sim-rendering.md) |
| Final output QA | [isaac-sim-validator](isaac-sim-validator.md) |

## Upstream source examples

| Script | Purpose | Arguments |
|---|---|---|
| [`scripts/control_loop.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/control_loop.py) | Control loop | see script --help |
| [`scripts/cumotion_setup.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/cumotion_setup.py) | Cumotion setup | see script --help |
| [`scripts/frames_and_grasping.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/frames_and_grasping.py) | Frames and grasping | see script --help |
| [`scripts/inspect_scene.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/inspect_scene.py) | Inspect scene | see script --help |
| [`scripts/phase_machine.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/phase_machine.py) | Phase machine | see script --help |
| [`scripts/world_binding.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/world_binding.py) | World binding | see script --help |

## Scope Rules

- Use current experimental Isaac Sim APIs. Do not add Cortex, deprecated
  manipulator, or legacy `omni.isaac.*` compatibility unless the user asks for
  migration. `RobotState` and the world-binding substrate live in
  `isaacsim.robot_motion.experimental.motion_generation`; the non-experimental
  `isaacsim.robot_motion.motion_generation` is deprecated (still shipped under
  `extsDeprecated`, not removed).
- Prefer interactive-session iteration (Jupyter kernel in the Antioch session;
  see [agentic-simulation](../../agentic-simulation/SKILL.md)). A standalone
  script or managed scenario can be the deliverable shape, but validate the
  scene logic, frame alignment, and phase gates interactively first.
- Do not trust shell exit code alone. Inspect stdout/logs for tracebacks,
  `RuntimeError`, explicit pass/fail JSON, and phase completion.
- Keep one-off probes in `/tmp`. Only keep reusable, domain-specific helpers
  in the project.
- Scripts and scenarios run in the remote service; there is no local window.
  Gate frame capture / video encoding behind explicit opt-in flags (for
  example `--render OUTPUT.MP4`), and never capture frames or encode a video
  unless asked.

## Workflow

1. Inspect robot prim path, DOF/link names, controller tool frames, object
   pose/AABB, physics schemas, and collision APIs.
2. Define the task frame before tuning: tool frame, physical grasp/action
   point, object semantic axis, and target axis.
3. Build world binding and controller ([world-binding](motion-generation-references-world-binding.md),
   [cumotion](motion-generation-references-cumotion.md)).
4. Run an explicit phase machine ([control-loop](motion-generation-references-control-loop.md)) with target,
   reset policy, convergence predicate, timeout, and trace line per phase.
5. Validate measured outputs from the actual run. For dynamic manipulation,
   use [manipulation-ik](manipulation-ik.md) and [physics-simulation](physics-simulation.md) for grasp/contact gates.

## References And Helpers

- [workflow](motion-generation-references-workflow.md): task architecture and phase-machine shape.
- [world-binding](motion-generation-references-world-binding.md): obstacles and robot-root transforms
  (`SceneQuery`, `ObstacleStrategy`, `WorldBinding`, `synchronize_transforms`).
- [control-loop](motion-generation-references-control-loop.md): `RobotState`/`JointState`/`SpatialState` builders
  and the `reset`/`forward` step that applies joint targets.
- [frames-and-grasping](motion-generation-references-frames-and-grasping.md): tool-frame offsets, axis contracts, and
  gripper targeting.
- [cumotion](motion-generation-references-cumotion.md): cuMotion-specific controller (`RmpFlowController`,
  supported robots, cspace params, RMPflow failure modes).
- [`scripts/inspect_scene.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/inspect_scene.py): interactive-session scene inspection (optional cuMotion
  probe via the `supported_robot` arg).
- [`scripts/cumotion/standalone_demo_template.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/cumotion/standalone_demo_template.py): copy-adapt standalone cuMotion
  scaffold.
- [`scripts/control_loop.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/control_loop.py): name-addressed `RobotState` builders and joint-target
  application.
- [`scripts/world_binding.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/world_binding.py): cuMotion obstacle discovery and per-frame world sync.
- [`scripts/cumotion_setup.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/cumotion_setup.py): supported-robot RMPflow controller construction.
- [`scripts/frames_and_grasping.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/frames_and_grasping.py): tool/contact offset conversion helpers.
- [`scripts/phase_machine.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/phase_machine.py): reusable manipulation phase labels.

For live-session helpers (asset verification, debug views, viewport video), use the interactive workflows in [agentic-simulation](../../agentic-simulation/SKILL.md).

## Controller wiring (quick rules)

Full code in `references/world-binding.md` + `references/control-loop.md`;
controller construction in `references/cumotion.md`. The non-negotiables:

- Build joint and site state by name, not by index assumptions
  (`robot.dof_names`, the controller's reported tool/site frames).
- Call `controller.reset(estimated, setpoint, t=0.0)` before the first
  `forward()` and after intentional target discontinuities.
- Pass controller clock time into `forward(...)`; do not pass `dt`.
- Apply positions, velocities, and efforts from the desired joint state when
  present.
- Synchronize the world binding every control frame
  (`update_world_to_robot_root_transforms(...)` then `synchronize_transforms()`).
- Keep the grasped object out of the tracked obstacle set, or the controller
  plans around it.
- For simple reactive obstacle avoidance, bind obstacles through
  `CumotionWorldInterface` / `WorldBinding` and keep the transport phase in the
  `RmpFlowController.forward()` loop. Do not switch to graph planning or author
  bypass/clearance targets unless the demo explicitly needs global planning.
- Tune obstacle inflation in small steps; large clearance buffers can improve
  avoidance but break grasp/place convergence.
- Log requested planner clearance and measured runtime clearance separately;
  they are not the same.
- Start with example-scale c-space/posture weights; large bias can hide weak
  task-space motion.
- For rendered demos, validate the recorder-enabled run, not only a non-captured
  metrics run.

## Mobile-Base Controller Pattern

For mobile bases, route planning and wheel geometry to
[navigation-primitives](navigation-primitives.md). Use this skill only for `RobotState` / controller
integration, and apply joint velocities/efforts/positions rather than writing
root transforms as the success path for a physics run.

## Frame Discipline

Most failures are frame errors. Log these separately (see
`references/frames-and-grasping.md`):

- controller tool frame, such as `tool0` or `wrist_3_link`
- desired and measured tool local `+Z` axis
- visible fingertip midpoint or suction tip
- task grasp point on the object
- object origin, center of mass, and AABB center
- object semantic axis and final target axis
- target pose sent to the controller

Command the tool pose that makes the physical grasp point land on the object
grasp point. If the controller drives a flange/tool frame but the gripper
contacts at a fingertip or suction cup, calibrate the tool-local offset in the
approach orientation and freeze it through close/lift.

Before coding a flip or placement task, prove the final pose is geometrically
feasible for the grasp. If a requested final pose requires an upward-facing
gripper or below-surface wrist, change the grasp strategy before tuning
controller gains or timeouts.

## Canonical Sources

Scripts (`source/standalone_examples/tutorials/manipulation/`, all use the `mg` substrate):

- `tutorial_9_arm_trajectory.py`, `tutorial_9_follow_target.py`
- `tutorial_9_pick_place_cumotion.py` (cuMotion), `tutorial_9_pick_place_pink.py` (PINK)

Docs (general API first, then implementations):

- `docs/isaacsim/robot_motion_experimental/index.rst`: framework overview
- `docs/isaacsim/motion_generation/{index,scene_interaction,trajectory_planning,mobile_robot_control_example}.rst`
- `docs/isaacsim/cumotion/index.rst`, `docs/isaacsim/pink/index.rst`
