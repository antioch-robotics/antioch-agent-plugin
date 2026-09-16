# cuMotion (RMPflow controller)

Adapted from NVIDIA [`motion-generation/references/cumotion.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/references/cumotion.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

cuMotion-specific reference for `isaacsim.robot_motion.cumotion`. The generic substrate
it plugs into lives in sibling references: obstacles / world binding in
[world-binding](motion-generation-references-world-binding.md), the `RobotState`
control loop in [control-loop](motion-generation-references-control-loop.md),
tool-frame discipline in
[frames-and-grasping](motion-generation-references-frames-and-grasping.md).

## Imports

In standalone scripts, import the controller after simulation startup
(`antioch.start_simulation()` or the managed scenario start).
Use [`scripts/cumotion_setup.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/cumotion_setup.py) for the supported-robot
loader and controller construction.

`CumotionWorldInterface` is the cuMotion implementation of the `world_interface` that
`mg.WorldBinding` requires (see
[world-binding](motion-generation-references-world-binding.md)).

## Controller construction

Call `build_rmpflow_controller(robot, world_binding, robot_name)` from
[`scripts/cumotion_setup.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/cumotion_setup.py). It returns the controller,
available tool frames, and selected tool frame.

Do not hard-code a tool frame until inspection proves it. Common UR10 configs expose
`tool0` or `wrist_3_link`, while the visible physical grasp point may be elsewhere (see
[frames-and-grasping](motion-generation-references-frames-and-grasping.md)).

Once constructed, drive it with the generic loop in
[control-loop](motion-generation-references-control-loop.md).

## Common RMPflow failures

- `forward()` never converges: wrong tool frame, wrong quaternion, stale target, or no
  reset after a target jump.
- robot moves but the object target is offset: controlling the flange/tool while
  measuring the fingertip or suction point (see
  [frames-and-grasping](motion-generation-references-frames-and-grasping.md)).
- object blocks the path unexpectedly: the grasped object is still in the world binding
  as an obstacle (see [world-binding](motion-generation-references-world-binding.md)).
- erratic motion after a phase transition: target changed discontinuously without
  `reset(...)`.
- gripper joint command ignored: gripper DOF index is wrong or resolved before
  physics / articulation initialization.
- FK rotation assumption fails: `CumotionRobot.kinematics.pose(...).rotation` can expose
  `matrix()` without a `.quaternion` property. Derive WXYZ from `rotation.matrix()` or
  inspect the current build's rotation API before using it.
