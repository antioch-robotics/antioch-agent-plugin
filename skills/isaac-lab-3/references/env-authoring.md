# Environment authoring — Isaac Lab 3.0 on Antioch

End-to-end pattern for a new `ManagerBasedRLEnvCfg` task. Every API name here
is verified against the Isaac Lab 3.0.0-beta2 typing stubs. Two Antioch rules
shape the whole file:

- **Lazy config definitions.** `isaaclab*` can never be imported at module
  scope, so every config class is defined inside a function — the
  `@configclass` decorators and `mdp` references only resolve once the
  scenario runs remotely.
- **Numbers out, not prints.** A run is judged by the metrics it returns;
  emit episode return / termination rate as run results.

Structure: one `build_env_cfg()` that returns the env config, plus a run
loop that boots, rolls out, and reports.

## The full config

`InteractiveSceneCfg` is the container; assets land as attributes and are
addressed later by attribute name (`SceneEntityCfg("robot")`). A ground
plane plus one articulation is the minimal useful scene. Observation,
action, reward, termination, and event managers each get their own
configclass:

```python
def build_env_cfg():
    from isaaclab.assets import ArticulationCfg, AssetBaseCfg
    from isaaclab.actuators import ImplicitActuatorCfg
    from isaaclab.envs import ManagerBasedRLEnvCfg, mdp
    from isaaclab.managers import EventTermCfg as EventTerm
    from isaaclab.managers import ObservationGroupCfg as ObsGroup
    from isaaclab.managers import ObservationTermCfg as ObsTerm
    from isaaclab.managers import RewardTermCfg as RewTerm
    from isaaclab.managers import SceneEntityCfg
    from isaaclab.managers import TerminationTermCfg as TermTerm
    from isaaclab.scene import InteractiveSceneCfg
    from isaaclab.sim import GroundPlaneCfg, SimulationCfg, UsdFileCfg
    from isaaclab.utils.configclass import configclass

    ROBOT_USD = "/path/to/robot.usd"  # keep the asset in the Antioch project

    @configclass
    class TaskSceneCfg(InteractiveSceneCfg):
        """Scene: ground plane + one articulation, replicated across envs."""

        ground = AssetBaseCfg(prim_path="/World/ground", spawn=GroundPlaneCfg())
        robot: ArticulationCfg = ArticulationCfg(
            prim_path="{ENV_REGEX_NS}/Robot",
            spawn=UsdFileCfg(usd_path=ROBOT_USD),
            init_state=ArticulationCfg.InitialStateCfg(pos=(0.0, 0.0, 1.0), joint_pos={".*": 0.0}),
            actuators={"all": ImplicitActuatorCfg(joint_names_expr=[".*"], stiffness=100.0, damping=10.0)},
        )

    @configclass
    class ActionsCfg:
        joint_pos = mdp.JointPositionActionCfg(asset_name="robot", joint_names=[".*"], scale=0.5, use_default_offset=True)

    @configclass
    class ObservationsCfg:
        @configclass
        class PolicyCfg(ObsGroup):
            joint_pos = ObsTerm(func=mdp.joint_pos_rel)
            joint_vel = ObsTerm(func=mdp.joint_vel_rel)
            base_lin_vel = ObsTerm(func=mdp.base_lin_vel)
            base_ang_vel = ObsTerm(func=mdp.base_ang_vel)
            projected_gravity = ObsTerm(func=mdp.projected_gravity)

        policy: PolicyCfg = PolicyCfg()

    @configclass
    class RewardsCfg:
        alive = RewTerm(func=mdp.is_alive, weight=1.0)
        upright = RewTerm(func=mdp.flat_orientation_l2, weight=-2.0)
        joint_vel = RewTerm(func=mdp.joint_vel_l2, weight=-1e-3)
        torques = RewTerm(func=mdp.joint_torques_l2, weight=-1e-5)

    @configclass
    class TerminationsCfg:
        time_out = TermTerm(func=mdp.time_out, time_out=True)
        fell = TermTerm(func=mdp.root_height_below_minimum, params={"minimum_height": 0.3})

    @configclass
    class EventsCfg:
        reset_base = EventTerm(
            func=mdp.reset_root_state_uniform, mode="reset", params={"pose_range": {"x": (-0.5, 0.5), "y": (-0.5, 0.5)}, "velocity_range": {}}
        )

    cfg = ManagerBasedRLEnvCfg(
        decimation=4,  # policy at 30 Hz against a 120 Hz sim
        episode_length_s=20.0,
        scene=TaskSceneCfg(num_envs=512, env_spacing=4.0),
        observations=ObservationsCfg(),
        actions=ActionsCfg(),
        rewards=RewardsCfg(),
        terminations=TerminationsCfg(),
        events=EventsCfg(),
    )
    cfg.sim = SimulationCfg(dt=1.0 / 120.0)
    return cfg
```

Notes on the pieces:

- `{ENV_REGEX_NS}` is the per-env namespace token Isaac Lab expands per
  clone; scene entities are replicated under it.
- Observation functions take `(env, asset_cfg=SceneEntityCfg("robot"))` by
  default — `SceneEntityCfg("robot")` resolves to the scene attribute named
  `robot`, so the names must match.
- `TermTerm(time_out=True)` marks the term as a truncation (time limit)
  rather than a terminal state; leave it `False` for real failures like
  falling.
- `EventsCfg` replaces the default event manager, so keep a reset-mode term
  (here `reset_root_state_uniform`) or the envs never re-randomize. Omit
  `events=` entirely to get the default `reset_scene_to_default`.

## Gym registration

Register once, at remote-run time (still inside a function — the entry point
string keeps the import lazy):

```python
def register_task() -> None:
    import gymnasium as gym

    gym.register(
        id="MyOrg-Task-v0", entry_point="isaaclab.envs:ManagerBasedRLEnv", disable_env_checker=True, kwargs={"env_cfg_entry_point": f"{__name__}:build_env_cfg"}
    )
```

`env_cfg_entry_point` accepts a callable or a `module:attr` string; the
string form defers evaluation until the env is constructed on the machine.
RL libraries (rl_games, rsl_rl, skrl) register the same task id with their
own `entry_point` and an agent config kwarg — that wiring is the library's
convention, not Isaac Lab's.

## Run loop (reset / step)

Direct construction skips the registry and is the fastest smoke test. The
loop below is also what a no-training sanity scenario looks like on Antioch:

```python
def run_task(num_steps: int = 2000) -> dict:
    import torch
    from isaaclab.envs import ManagerBasedRLEnv

    env = ManagerBasedRLEnv(cfg=build_env_cfg())
    obs, _ = env.reset()
    episode_return = torch.zeros(env.num_envs, device=env.device)
    completed = []
    for _ in range(num_steps):
        action = 0.2 * torch.randn(env.action_space.shape, device=env.device)
        obs, rew, terminated, truncated, extras = env.step(action)
        episode_return += rew
        done = terminated | truncated
        if done.any():
            completed.extend(episode_return[done].tolist())
            episode_return[done] = 0.0
    env.close()
    return {"episodes_completed": len(completed), "mean_episode_return": sum(completed) / max(len(completed), 1)}
```

`env.step` returns `(obs, reward, terminated, truncated, extras)` — the
Gymnasium five-tuple. Random actions are only a boot smoke test; real runs
put a policy in that slot.

## Antioch framing

Wrap `run_task` in a thin scenario and dispatch with `antioch scenario run` /
`antioch run` (dispatch mechanics are the `antioch-platform` skill's job).
The decorator requires the first parameter annotated `antioch.ScenarioRun`
with no default, and a scenario's return value is discarded — results reach
the run's record through `run.add_results(...)`, not a return:

```python
@antioch.scenario()
def my_task(run: antioch.ScenarioRun, num_steps: int = 2000) -> None:
    run.add_results(run_task(num_steps))
```

A scenario body needs nothing else — the runner boots Kit before calling
it. A one-off `antioch run` script owns the lifecycle instead: call
`antioch.boot()` before `run_task`, which constructs `ManagerBasedRLEnv`
and needs a live SimulationApp.

- Antioch's managed Lab launcher always boots through Kit (`AppLauncher`
  → `SimulationApp`), so nothing here touches `SimulationApp` directly and
  the real per-run choice is PhysX vs Newton, not Kit vs kit-less.
- Scale `num_envs` up only after the 512-env config holds: watch GPU memory
  per step and cut `num_envs` before debugging anything else on OOM. For
  vision-in-the-loop tasks, budget cameras too — roughly 512 cameras per
  24 GB GPU at small resolutions.
- Judge stability over multi-second episodes (`episode_length_s` of 20 at
  120 Hz sim / 30 Hz policy is 600 policy steps). Debug curves in order:
  flat-zero reward → read the per-term reward and per-term termination
  percentages in the env logs (a termination firing immediately or one
  reward term dominating); NaN loss → suspect the physics config first
  (joint/force limits, actuator gains, timestep, solver iterations), RL
  hyperparameters second.

## Robot assets: `isaaclab.sim.converters`

Lab's native import path for a robot URDF/MJCF is `UrdfConverter` /
`MjcfConverter` with their `*Cfg` classes — a programmatic wrapper over the
Isaac Sim importer that adds `make_instanceable` (default `True`, what
parallel-env RL wants) and a nested `JointDriveCfg` for drive type and PD
gains. Convert once, then reference the USD from the scene config. Isaac Sim
importer field detail (fix_base tri-state, density, gain overrides): the
`isaac-sim-6` skill's assets references.
