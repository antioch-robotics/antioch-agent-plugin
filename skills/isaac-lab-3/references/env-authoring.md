# Environment authoring and rollout

Lab environment structure, manager terms, reset/step behavior, task
registration, RL adapters, and demonstration datasets. The parent skill owns
startup, pins, quaternion order, and array access.

## Choose the environment boundary

A manager-based task puts scene, actions, observations, rewards,
terminations, and events in configuration; a direct environment owns them in
code. Match the existing project. Define config classes inside a post-startup
factory and use a shipped, validated asset/config rather than a placeholder
path. Resolve scene names and every `SceneEntityCfg` selector against actual
ordered joint/body names, including unrestricted `slice(None)`.

## State transition and reset

Not every `.data.*` field is an array. Convert WXYZ only at an Isaac Sim
boundary. `SceneEntityCfg.joint_ids/body_ids` may be `slice(None)`; resolve
selectors against the articulation's ordered names. Older state-write methods
forward with a deprecation warning at this pin.

`ProxyArray.torch` is a cached zero-copy view and `.warp` exposes the Warp
array. Clone a historical sample, and re-access the data property after a
full reset or buffer recreation, especially on Newton, since an old wrapper
may refer to replaced storage.

In a manually owned scene loop, write controls, step, then call
`scene.update(sim.get_physics_dt())`; a zero timestep is not a refresh. Do
not add duplicate updates around a managed environment. Check batch axes,
environment origins, frames, and joint order before computing errors.


Specify observation schema/history/noise/frame; action shape/order/scale/limits,
actuator mode, and control period; reward terms, signs, weights, units, and
aggregation; success/failure terminations versus time-limit truncation; and a
reset for robot pose/velocity, joints, objects, sensors, and task buffers.

An events config replaces the selected event behavior. Resetting a base pose
does not reset joints or task state unless those terms are included; test a
reset after a disturbed episode, not only after construction.

`ManagerBasedRLEnv.step` returns `(observation, reward, terminated, truncated,
extras)`. Preserve the distinction for value bootstrapping and honor the
selected wrapper's autoreset contract.

## Small startup probe

A shipped task tests lifecycle and tensor shapes before changing a robot.
This probe checks finite rewards and that an episode completes; it is not a
trained-policy test.

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

## Registration and RL

Gym registration maps an environment ID to its class/config entry point. RL
agent configuration is additional registration data consumed by the training
adapter; it does not replace the environment entry point. Resolve the pinned
`module:attribute` or callable from task-registration examples.

At this pin, rsl-rl 5.x uses separate actor/critic configuration. Pass a
shipped task's runner config through
`handle_deprecated_rsl_rl_cfg(agent_cfg, installed_version)` before
`OnPolicyRunner`, as upstream `train_rsl_rl.py` does.

Wrap the environment with the installed RL library's supported adapter and
match its reset, observation/action, runner, and checkpoint contracts. Save
config and asset identity with a checkpoint and load it in a fresh evaluation
run.

For URDF/MJCF conversion, `isaaclab.sim.converters` wraps the import
pipeline; inspect its options rather than assuming importer defaults, and
validate the composed USD through the Isaac Sim asset workflow.

## Demonstrations and imitation

Teleoperation, demonstration replay, and Lab Mimic are separate from
Replicator image generation. Keep task identity, action frame/control mode,
observations, subtask boundaries, and success criteria consistent.

The pinned `scripts/tools/record_demos.py` writes successful episodes to
HDF5; optional MCAP is teleoperation debugging output, not the training
dataset. Keep attempted, failed, and accepted counts separate. Mimic requires
task-space actions and subtask annotations; convert joint-space
demonstrations as the documented pipeline requires.

The recording script's `AppLauncher` and input device path must match the
selected Antioch workflow; upstream keyboard, SpaceMouse, XR, or CloudXR
support does not prove remote input forwarding. Replay a sample through the
intended reader before scaling collection, and archive datasets/checkpoints
as run artifacts.

## Scaling and evaluation

Start with a small `num_envs` and increase it after the full reset/step loop works. Profile camera/render-product memory, scene state, observations/history, and optimizer state separately. Disabling rendering may help a state-only policy, but it changes the experiment if observations require images. Keep action rate, physics timestep, and decimation consistent while comparing throughput.

Evaluate a checkpoint in a fresh run using the intended observation normalization, action scaling, actuator mode, and inference device. Record completed episode counts, success/failure criteria, return distributions, seeds, and environment configuration. Separate time-limit truncation from task failure. Compare with a baseline and hold out seeds or conditions when measuring generalization; a finite loss or saved checkpoint alone is not policy evidence.

Use [scenario cases](../../scenario-design/SKILL.md#inputs-and-execution-policy) and [suite selection](../../antioch-platform/references/suites.md) to preserve evaluations, [telemetry](../../scenario-design/references/telemetry.md) for review frames and metrics, and [asset publication](../../antioch-platform/references/assets.md#publish) for reusable datasets or checkpoints. Return to [Isaac Lab](../SKILL.md#load-the-part-you-need) for startup and native API contracts.

For reusable robots, datasets, and checkpoints, search [the asset catalog](../../antioch-platform/references/assets.md) before creating new content. For native backend boundaries, use [Isaac Sim physics](../../isaac-sim-6/references/physics-simulation.md), then confirm the specific Lab wrapper through [research](../../antioch-research/SKILL.md).
