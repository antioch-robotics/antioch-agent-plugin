# World binding (obstacles and robot-root transforms)

Adapted from NVIDIA [`motion-generation/references/world-binding.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/references/world-binding.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

Generic `isaacsim.robot_motion.experimental.motion_generation` (`mg`) substrate for
turning stage prims into motion-generation obstacles. Controller-agnostic: any
motion-generation controller consumes the same `WorldBinding`. The one
controller-specific piece is the concrete `world_interface` (for cuMotion,
`CumotionWorldInterface`; see [cumotion](motion-generation-references-cumotion.md)).

## Build the binding

Use [`scripts/world_binding.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/world_binding.py). Call
`build_world_binding(robot, robot_path, excluded_prim_paths)` after articulation
initialization; include the manipulated object in `excluded_prim_paths` while grasped.

If `WorldBinding.initialize()` fails in `validate_ancestor_scaling()` on a staged
asset with authored scale stacks, narrow or empty `tracked_prims` and disclose
the fallback in demo evidence. Do not report obstacle-aware motion when cuMotion
is running without those tracked obstacles.

## Synchronize each frame

Call `update_world_to_robot_root_transforms(robot.get_world_poses())` then
`synchronize_transforms()` every control frame when obstacles or the robot root move.
Shape properties and collision-enabled state are read during initialization and are not
synchronized at runtime. The per-frame transform sync is wired into the step in
[control-loop](motion-generation-references-control-loop.md).

## Exclude the grasped object

Once the object is grasped, keep it out of the tracked obstacle set (e.g. in
`exclude_prim_paths`). A grasped object left in the world binding becomes an obstacle
the controller plans around, so the arm refuses to move or detours unexpectedly.
