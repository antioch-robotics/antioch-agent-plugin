# Motion Generation

Adapted from NVIDIA [`motion-generation/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/SKILL.md) (Apache-2.0).

[Isaac Sim task index](../SKILL.md#domain-references) · [Antioch startup](../../antioch-platform/references/simulation-code.md)

Owns the generic `isaacsim.robot_motion.experimental.motion_generation` (`mg`)
substrate and the phase-machine workflow for obstacle-aware end-effector
motion. The motion-generation controller is pluggable: cuMotion
(`isaacsim.robot_motion.cumotion`) is the reference implementation here. PINK and
Lula are also motion-generation controllers but are not yet documented in this
skill; see the stack-selection table in [manipulation-ik](manipulation-ik.md) to choose.

Load related guidance for the next part of the task:

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
| [`scripts/control_loop.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/control_loop.py) | Name-addressed state builders and joint-target application | inspect source before use |
| [`scripts/cumotion_setup.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/cumotion_setup.py) | Supported-robot controller construction | inspect source before use |
| [`scripts/frames_and_grasping.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/frames_and_grasping.py) | Tool/contact offset conversion | inspect source before use |
| [`scripts/inspect_scene.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/inspect_scene.py) | Robot, frame, obstacle, and optional cuMotion inspection | inspect source before use |
| [`scripts/phase_machine.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/phase_machine.py) | Reusable manipulation phase labels | inspect source before use |
| [`scripts/world_binding.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/world_binding.py) | Obstacle discovery and per-frame synchronization | inspect source before use |

## Execution

Use `isaacsim.robot_motion.experimental.motion_generation` for the current
`RobotState` and world-binding APIs. Research a controller's pinned surface
before copying an older example. Jupyter is useful for inspecting frames and
phase transitions, but use the execution path that fits the task. Capture
images when needed to verify it, and keep costly video generation bounded.

The non-experimental `isaacsim.robot_motion.motion_generation` is deprecated but still ships under `extsDeprecated`; use the current experimental surface for new code. Inspect the run's logs, phase completion, checks, and final measured state as well as its exit code. A process exiting cleanly does not prove that the controller reached its goal.

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

## References and helpers

- [workflow](motion-generation-references-workflow.md): task architecture and phase-machine shape.
- [world-binding](motion-generation-references-world-binding.md): obstacles and robot-root transforms
  (`SceneQuery`, `ObstacleStrategy`, `WorldBinding`, `synchronize_transforms`).
- [control-loop](motion-generation-references-control-loop.md): `RobotState`/`JointState`/`SpatialState` builders
  and the `reset`/`forward` step that applies joint targets.
- [frames-and-grasping](motion-generation-references-frames-and-grasping.md): tool-frame offsets, axis contracts, and
  gripper targeting.
- [cumotion](motion-generation-references-cumotion.md): cuMotion-specific controller (`RmpFlowController`,
  supported robots, cspace params, RMPflow failure modes).

For a standalone scaffold, inspect the upstream [cuMotion demo template](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/cumotion/standalone_demo_template.py). Replace its app launch with Antioch startup; keep its state and controller wiring.

## Controller wiring (quick rules)

Use the linked world-binding, control-loop, and controller references above.
Check these boundaries:

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
[frames and grasping](motion-generation-references-frames-and-grasping.md)):

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

Tutorials at the pinned source revision, all using the `mg` substrate:

- [Arm trajectory](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/standalone_examples/tutorials/manipulation/tutorial_9_arm_trajectory.py) and [follow target](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/standalone_examples/tutorials/manipulation/tutorial_9_follow_target.py)
- Pick and place with [cuMotion](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/standalone_examples/tutorials/manipulation/tutorial_9_pick_place_cumotion.py) or [PINK](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/standalone_examples/tutorials/manipulation/tutorial_9_pick_place_pink.py)

The public source tree has these extension overviews and API references:

- Experimental framework: [overview](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/extensions/isaacsim.robot_motion.experimental.motion_generation/docs/Overview.md) and [API reference](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/extensions/isaacsim.robot_motion.experimental.motion_generation/docs/api.rst)
- [cuMotion overview](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/extensions/isaacsim.robot_motion.cumotion/docs/Overview.md) and [PINK overview](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/extensions/isaacsim.robot_motion.pink/docs/Overview.md)
- [Motion-generation example series](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/examples/series/motion_generation/README.md), including [pick and place](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/examples/series/motion_generation/pick_place/README.md)

The original skill also names `docs/isaacsim/...` pages that are absent from this public revision. Use the linked extension documentation and tutorials above rather than treating those paths as downloadable files.
