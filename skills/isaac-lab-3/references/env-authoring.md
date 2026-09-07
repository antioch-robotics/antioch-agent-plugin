# Environment authoring and rollout

Use this when building a Lab environment, changing manager terms, registering
a task, debugging reset/step behavior, or preparing demonstration datasets.
The parent skill owns startup,
runtime pins, quaternion ordering, and array access.

## Choose the environment boundary

A manager-based task separates scene, actions, observations, rewards,
terminations, and events into configuration. A direct environment owns those
operations in code. Start with the model that fits the existing project;
do not introduce managers solely to wrap a small working direct task.

Define config classes inside a factory so local discovery does not import
the simulator. Use a shipped robot config or a validated asset rather than
a placeholder USD path presented as a runnable example.

The scene's asset names and `SceneEntityCfg` selectors must agree. Resolve
joint/body selectors against actual names, including the unrestricted
`slice(None)` case.

## The state transition

For an RL environment, specify:

1. Inputs: observation schema, history, noise, and frame.
2. Actions: shape, joint order, scale, limits, actuator mode, and control period.
3. Reward: each term, sign, weight, units, and aggregation.
4. Termination: actual failures/success conditions versus time-limit truncation.
5. Reset: robot pose and velocity, joints, objects, sensors, and task state.

An events config replaces the selected event behavior; a base-pose reset alone
does not reset joint state or task buffers. Retain a complete default reset
or supply all required reset terms. Test a reset after a disturbed episode,
not just after initial construction.

`ManagerBasedRLEnv.step` returns observation, reward, terminated, truncated,
and extras. Distinguish termination from truncation when bootstrapping value
targets. Respect the selected environment/wrapper's autoreset contract.

## A small recorded startup probe

This uses the shipped cart-pole config, not a generic robot with guessed
drives. It checks finite rewards and completed episodes; it does not certify a
trained balancing policy. The scenario runner starts Lab before the body.

```python
import antioch


@antioch.scenario(tags=["smoke"], capture=False)
def cartpole_rollout(run: antioch.ScenarioRun, steps: int = antioch.param(600, ge=1), seed: int = 1) -> None:
    import torch
    from isaaclab.envs import ManagerBasedRLEnv
    from isaaclab_tasks.manager_based.classic.cartpole.cartpole_env_cfg import CartpoleEnvCfg

    cfg = CartpoleEnvCfg()
    cfg.scene.num_envs = 4
    cfg.seed = seed
    env = ManagerBasedRLEnv(cfg=cfg)
    try:
        env.reset()
        episode_return = torch.zeros(env.num_envs, device=env.device)
        action = torch.zeros(env.action_space.shape, device=env.device)
        completed = []
        finite_rewards = True
        for _ in range(steps):
            _, reward, terminated, truncated, _ = env.step(action)
            finite_rewards = finite_rewards and bool(torch.isfinite(reward).all())
            episode_return += reward
            done = terminated | truncated
            completed.extend(episode_return[done].tolist())
            episode_return[done] = 0
    finally:
        env.close()

    run.check("finite rewards", finite_rewards)
    run.check("episodes completed", bool(completed), detail=f"{len(completed)} completed episodes")
    run.add_result("episodes_completed", len(completed))
    if completed:
        run.add_result("mean_episode_return", sum(completed) / len(completed))
```

Inspect observations, actions, and physical state too when evaluating a new
task. Adapt the checks to the requested behavior rather than calling this
smoke probe a success test.

## Registration and RL integration

Gym registration maps an environment ID to an environment class and config
entry points. RL agent configs are additional registration kwargs consumed
by their training adapters; they do not require replacing the environment's
entry point with an RL library.

Use the pinned task-registration and training examples to resolve a
`module:attribute` config entry or callable. Direct construction with `cfg=`
is useful for a small probe; a registered task fits shared training tools.

Wrap the environment with the installed RL library's supported adapter.
Match its observation/action and reset semantics, runner config, and
checkpoint format. Save a checkpoint with enough config and asset identity
to load it into a fresh evaluation run.

For URDF/MJCF conversion, Lab's `isaaclab.sim.converters` wraps the import
pipeline. Inspect the chosen converter's options and output rather than
assuming identical drive or instanceability settings across importers.
The Isaac Sim asset reference owns composed-USD and physical validation.

## Demonstrations and imitation learning

Human teleoperation, demonstration replay, and Isaac Lab Mimic are a separate
workflow from Replicator image generation. Retrieve the pinned Lab recording,
replay, Mimic annotation/generation, and policy-training examples together.
Keep task identity, action frame/control mode, observations, subtask boundaries,
and success criteria consistent through that sequence.

The pinned `scripts/tools/record_demos.py` exports successful episodes to HDF5.
Its optional MCAP recording is teleoperation debugging output, not the training
dataset. Check exported episode counts and replay a sample; preserve attempted,
failed, and accepted counts separately rather than calling every attempt a
successful demonstration. Mimic needs task-space actions and subtask
annotations; a joint-space demonstration needs the documented conversion.

The upstream recording script owns an `AppLauncher`. Adapt startup and lazy
imports for the selected Antioch path; do not create a second app inside a
scenario. Confirm a working remote input path before promising keyboard,
SpaceMouse, or XR collection. Upstream device support does not prove Antioch
device forwarding or CloudXR availability, and does not authorize host-device
or network changes. Existing datasets can support an offline starting point
when human input is unavailable.

Archive completed datasets and checkpoints as run artifacts. Validate them
through the intended reader in a separate run before scaling collection or
training; report unrun input, replay, and policy-evaluation checks explicitly.
