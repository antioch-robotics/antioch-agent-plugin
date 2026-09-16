# Environment authoring and rollout

Use this for Lab environment structure, manager terms, reset/step behavior,
task registration, RL adapters, and demonstration datasets. The parent skill
owns startup, pins, quaternion order, and array access.

## Choose the environment boundary

A manager-based task puts scene, actions, observations, rewards, terminations,
and events in configuration. A direct environment owns them in code. Match the
existing project; do not wrap a small direct task in managers just for shape.
Define config classes inside a post-startup factory and use a shipped,
validated asset/config rather than a placeholder path.

Resolve scene names and every `SceneEntityCfg` selector against actual ordered
joint/body names, including unrestricted `slice(None)`.

## State transition and reset

Specify observation schema/history/noise/frame; action shape/order/scale/limits,
actuator mode, and control period; reward terms, signs, weights, units, and
aggregation; success/failure terminations versus time-limit truncation; and a
reset for robot pose/velocity, joints, objects, sensors, and task buffers.

An events config replaces the selected event behavior. Resetting a base pose
does not reset joints or task state unless those terms are included. Test a
reset after a disturbed episode, not only after construction.

`ManagerBasedRLEnv.step` returns `(observation, reward, terminated, truncated,
extras)`. Preserve the distinction for value bootstrapping and honor the
selected wrapper's autoreset contract.

## Small startup probe

Use a shipped task to test lifecycle and tensor shapes before changing a
robot. This probe checks finite rewards and that an episode completes; it is
not a trained-policy test.

```python
import antioch


@antioch.scenario(tags=["smoke"], capture=False)
def cartpole_rollout(run: antioch.ScenarioRun, steps: int = antioch.param(600, ge=1), seed: int = 1) -> None:
    import torch
    from isaaclab.envs import ManagerBasedRLEnv
    from isaaclab_tasks.manager_based.classic.cartpole.cartpole_env_cfg import CartpoleEnvCfg

    cfg = CartpoleEnvCfg()
    cfg.scene.num_envs, cfg.seed = 4, seed
    env = ManagerBasedRLEnv(cfg=cfg)
    try:
        env.reset()
        total = torch.zeros(env.num_envs, device=env.device)
        action = torch.zeros(env.action_space.shape, device=env.device)
        completed, finite = [], True
        for _ in range(steps):
            _, reward, terminated, truncated, _ = env.step(action)
            finite &= bool(torch.isfinite(reward).all())
            total += reward
            done = terminated | truncated
            completed.extend(total[done].tolist())
            total[done] = 0
    finally:
        env.close()
    run.check("finite rewards", finite)
    run.check("episodes completed", bool(completed), detail=f"{len(completed)} completed episodes")
    run.add_result("episodes_completed", len(completed))
    if completed:
        run.add_result("mean_episode_return", sum(completed) / len(completed))
```

For a real task, also inspect observations, actions, physical state, reset
completeness, and task-specific checks. Do not call this probe policy success.

## Registration and RL

Gym registration maps an environment ID to its class/config entry point. RL
agent configuration is additional registration data consumed by the training
adapter; it does not replace the environment entry point. Resolve the pinned
`module:attribute` or callable from task-registration examples.

Wrap the environment with the installed RL library's supported adapter and
match its reset, observation/action, runner, and checkpoint contracts. Save
config and asset identity with a checkpoint and load it in a fresh evaluation
run.

For URDF/MJCF conversion, `isaaclab.sim.converters` wraps the import pipeline;
inspect its options rather than assuming importer defaults. Validate the
resulting composed USD and physical asset through the Isaac Sim asset workflow.

## Demonstrations and imitation

Teleoperation, demonstration replay, and Lab Mimic are separate from
Replicator image generation. Retrieve matching pinned recording, replay,
Mimic annotation/generation, and policy-training examples. Keep task identity,
action frame/control mode, observations, subtask boundaries, and success
criteria consistent.

The pinned `scripts/tools/record_demos.py` writes successful episodes to HDF5;
optional MCAP is teleoperation debugging output, not the training dataset.
Keep attempted, failed, and accepted counts separate. Mimic requires task-space
actions and subtask annotations; convert joint-space demonstrations as the
documented pipeline requires.

The recording script's `AppLauncher` and input device path must match the
selected Antioch workflow. Upstream keyboard, SpaceMouse, XR, or CloudXR
support does not prove remote input forwarding. Validate dataset counts and
replay a sample through the intended reader before scaling collection or
training. Archive datasets/checkpoints as run artifacts and report unrun
checks.
