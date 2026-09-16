# Control loop (RobotState in, joint targets out)

Adapted from NVIDIA [`motion-generation/references/control-loop.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/references/control-loop.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

Generic motion-generation control contract: build an estimated `RobotState` and a
setpoint `RobotState`, call `reset(...)` then `forward(...)` each frame, and apply the
returned joint targets. Controller construction is controller-specific (cuMotion:
[cumotion](motion-generation-references-cumotion.md)); this loop is the same
regardless of controller.

## RobotState helpers

Use [`scripts/control_loop.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/control_loop.py): `estimated_state`,
`setpoint_state`, and `apply_desired_joint_state` keep the name-addressed state
and target application contract in one reusable module.

Build state by name, not by index assumptions: use `robot.dof_names` for the joint
space and the controller's reported site/tool frames for the site space.

`orientation` is a WXYZ quaternion. Confirm any Euler conversion helper returns WXYZ
before using it. In headless or sandboxed validation, prefer explicit quaternions for
canonical poses; some transform helper paths compile Warp kernels and may fail if the
Warp cache is not writable.

## Step

Use `synchronize_world_binding` from [`scripts/world_binding.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/world_binding.py)
before every `controller.forward(...)`, then apply its result with
`apply_desired_joint_state` from [`scripts/control_loop.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/control_loop.py).

Rules:

- Call `reset(estimated, setpoint, t=0.0)` before the first `forward()` and after any
  intentional target discontinuity (a phase jump). Skipping it produces erratic motion.
- Pass controller *clock time* into `forward(...)`, accumulated by `+= physics_dt`. Do
  not pass `dt`.
- Apply positions, velocities, and efforts from the desired joint state when present.
