# World binding (obstacles and robot-root transforms)

Adapted from NVIDIA [`motion-generation/references/world-binding.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/references/world-binding.md) (Apache-2.0).

[Isaac Sim task index](../SKILL.md#domain-references) · [Antioch startup](../../antioch-platform/references/simulation-code.md)

Generic `isaacsim.robot_motion.experimental.motion_generation` (`mg`) substrate for
turning stage prims into motion-generation obstacles. Controller-agnostic: any
motion-generation controller consumes the same `WorldBinding`. The one
controller-specific piece is the concrete `world_interface` (for cuMotion,
`CumotionWorldInterface`; see [cumotion](motion-generation-references-cumotion.md)).

## Build the binding

Use [`scripts/world_binding.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/world_binding.py). Call
`build_world_binding(robot, robot_path, excluded_prim_paths)` after articulation
initialization. Pass excluded objects through `excluded_prim_paths` before the helper initializes the binding. Changing a Python exclusion list after initialization does not remove an already tracked obstacle.

If `WorldBinding.initialize()` fails in `validate_ancestor_scaling()` on a staged
asset with authored scale stacks, narrow or empty `tracked_prims` and disclose
the fallback in demo evidence. Do not report obstacle-aware motion when cuMotion
is running without those tracked obstacles.

## Synchronize each frame

Call `update_world_to_robot_root_transforms(robot.get_world_poses())` then
`synchronize_transforms()` every control frame when obstacles or the robot root move.
Call `synchronize_properties()` when collision-enabled state or shape attributes change, or `synchronize()` to update both transforms and properties. Local-scale changes are not supported by property synchronization; rebuild the binding after changing a tracked object's scale. The per-frame transform sync is wired into the step in
[control-loop](motion-generation-references-control-loop.md).

## Exclude the grasped object

Exclude the object before initialization if it should never be a world obstacle. If it must remain an obstacle until grasp, rebuild the binding at grasp time with the object excluded, then synchronize the remaining obstacles and robot-root transform before planning again. Rebuild with the object included after release when the task needs it. A grasped object left in the world binding becomes an obstacle the controller plans around, so the arm can refuse to move or detour unexpectedly.
