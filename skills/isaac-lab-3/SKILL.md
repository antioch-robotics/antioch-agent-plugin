---
name: isaac-lab-3
description: >
  Guides authoring and debugging of Isaac Lab 3.0 environments, managers,
  tasks, and RL training code that runs remotely on Antioch GPU machines.
  Teaches the lazy-import invariant, XYZW quaternions, ProxyArray data access,
  the _index/_mask write_*_to_sim split, the PhysX vs Newton backend choice,
  and environment authoring patterns. Use proactively for ANY Isaac Lab
  task — authoring a manager-based env or task, adding observation, reward,
  or termination terms, training or tuning RL with rsl-rl or skrl, scaling
  num_envs, or porting 2.x Lab code — and whenever isaaclab* appears in an
  import or traceback. Not for plain Isaac Sim work (isaac-sim-6), dispatch
  and run history (antioch-platform), scenario verdict design
  (scenario-design), or API signatures — ground those with antioch-research.
---

# Isaac Lab 3 for Antioch

Isaac Lab 3.0.0-beta2 — still a beta: APIs can shift between releases, so pin
assumptions to the exact version on the machine and say so when an API looks
unstable. Built on Isaac Sim 6.0; the `isaac-sim-6` skill covers the
underlying stage, physics, and USD layer.

Code is authored locally (no simulator installed) and executed remotely on
Antioch GPU machines. How to dispatch runs and read back metrics/artifacts:
load the `antioch-platform` skill. This skill is
only about writing correct Isaac Lab code. Isaac runs on the remote machine;
simulation commands stream its GUI by default, and `--no-stream` runs
headlessly. Verify training through recorded curves, metrics, and artifacts,
not only through the viewport.

Queue workers run immutable images and do not apply development watch rules.
Antioch adds the submitted project tree to the simulation image before the
queue accepts it, whether `services.sim` names an image or a custom build.

## Research first

Before writing or debugging ANY code that touches an Isaac Lab, Isaac Sim,
Warp, or RL-runner surface, ground it with the antioch-research MCP:
`research_search` for signatures, config fields, defaults, and behavior
(`kind="source"` to localize the implementing code), and `research_open` to
read a whole vendor file. The corpora are pinned to the machine's exact
versions — Isaac Lab 3.0.0-beta2 docs + source, Isaac Sim 6.0.1, Warp 1.13.0,
rsl-rl v5.0.1, skrl, and cuRobo are all indexed — so RL-runner config formats
and beta-drift questions are researchable, not guessable. This skill orients;
research grounds.

## Running on Antioch — who boots Kit

Every run is one process holding one Kit, and on Lab engines Antioch boots it
through Isaac Lab's own `AppLauncher` (the sanctioned Lab launch path) — so
the AppLauncher-before-imports ordering is handled for you. Who starts it
depends on the entry point:

- **Scenarios** (`@antioch.scenario` + `antioch scenario run`): the runner
  boots before calling the function. Scenario bodies never call
  `antioch.boot()` and never create their own `AppLauncher` — a second one
  conflicts with the first.
- **Plain scripts** (`antioch run script.py`) and **notebooks**: the code
  owns the lifecycle. Call `antioch.boot()` once, before any `isaaclab*` or
  `isaacsim` import; the options are tabulated in the `isaac-sim-6` skill's
  "Running on Antioch" section and apply identically here — except the
  `physics_dt`/`render_dt` timing knobs, which only feed `antioch.world()`'s
  `World` construction and are therefore inert on Lab.

Handles into native state after boot: `antioch.stage()` (the current USD
stage, any engine), `antioch.application()` (the live `SimulationApp` when
Antioch booted it), `antioch.is_running()` / `antioch.engine()`. Note
`antioch.world()` is Isaac Sim only — on Lab it raises by design; the native
equivalent is `isaaclab.sim.SimulationContext`.

Everything past these handles is **native Isaac Lab** — envs, managers,
assets, sensors, and RL runners are exactly what upstream documents; Antioch
adds no wrapper layer.

## Rule #1 — lazy simulator imports (hard invariant)

`isaaclab*`, `isaacsim`, `omni`, `pxr`, and `carb` must NEVER be imported at
module scope — only inside function bodies or under
`if TYPE_CHECKING:`. Antioch discovers scenarios locally with no simulator
installed; a module-scope simulator import breaks discovery and fails loudly
at boot.

```python
# WRONG — crashes local discovery
from isaaclab.envs import ManagerBasedRLEnvCfg


# RIGHT — import inside the function that runs remotely
def build_env_cfg():
    from isaaclab.envs import ManagerBasedRLEnvCfg

    ...
```

`torch` is NOT one of the enforced roots — it won't break discovery. Defer
its import until after the app boots anyway, but for a different reason:
importing torch before Kit owns the CUDA context can hang Kit.

## 3.0 changes from 2.x (check these first in ported code)

| Area | 2.x | 3.0 |
|---|---|---|
| Quaternions | WXYZ | **XYZW globally** — opposite of Isaac Sim's USD convention |
| `.data.*` properties | `torch.Tensor` | **`ProxyArray` over a Warp array** — use `.torch` / `.warp` |
| `write_*_to_sim` | single method | split into `_index` / `_mask` variants (old single call still forwards with a deprecation warning) |
| Headless CLI | `--headless` | headless is default; `--viz` / `--visualizer` opts in (`--headless` still accepted, deprecated) |
| Tiled camera | `TiledCamera` class | folded into `Camera` (`TiledCamera` remains as a deprecated alias) |
| Physics backend | PhysX only | `isaaclab_physx` or `isaaclab_newton`, chosen per run |
| Physics config | `SimulationCfg.physx: PhysxCfg` from `isaaclab.sim` | `SimulationCfg.physics: PhysicsCfg`, and the concrete class moved to the backend package: `from isaaclab_physx.physics import PhysxCfg` |
| Runtime | older Python/torch | Python 3.12, PyTorch 2.10 |

## The quaternion trap (highest-frequency bug)

Isaac Lab 3.0 uses XYZW everywhere; USD/Isaac Sim underneath uses WXYZ; scipy
uses XYZW. Code that crosses the boundary (spawning from USD poses, reading
sim state, calling scipy) must convert deliberately:

```python
# USD WXYZ -> Isaac Lab XYZW
xyzw = [wxyz[1], wxyz[2], wxyz[3], wxyz[0]]
```

The identity quaternion moved with the convention: it is now `(0, 0, 0, 1)`.
A leftover `rot=(1.0, 0.0, 0.0, 0.0)` from 2.x-era code compiles fine but
rotates the asset — treat any WXYZ-looking literal in ported code as a bug
until proven otherwise.

Never pass a quaternion across the Isaac Sim ↔ Isaac Lab boundary without
stating which convention it is in. A 90°-off or tumbling asset is almost
always this.

## Working with .data.* properties (ProxyArray)

Every `.data.*` property (root state, joint state, sensor buffers) returns a
`ProxyArray` — a zero-copy wrapper over a Warp array. Its explicit accessors
are `.torch` (cached zero-copy `torch.Tensor` view) and `.warp` (the
underlying `wp.array`). Indexing, arithmetic, and common torch functions
forward to the torch view through a deprecation bridge that warns once per
process; instance methods (`clone()`, `.numpy()`, …) are not forwarded at all
— go through `.torch` explicitly so the convention is visible:

```python
def inspect(robot) -> None:
    import torch

    root_pos = robot.data.root_pos_w.torch  # torch.Tensor, zero-copy
    root_pos_wp = robot.data.root_pos_w.warp  # wp.array, for kernel interop
    root_pos_cpu = root_pos.cpu().numpy()  # only when CPU NumPy is genuinely required
```

Stay on `.torch` for math and `.warp` for kernels; don't route through NumPy
in hot loops — batch the read and convert once, at the boundary.

**Never call `articulation.update(0.0)`.** The buffers behind every `.data.*`
property are timestamped, and `update(dt)` advances that timestamp by `dt`.
With `dt=0` the buffers still consider themselves fresh, so every readback
silently returns the values from the last real update — usually the post-reset
pose. There is no warning and no error. A control loop written this way reports
a stationary arm at hundreds of millimetres of tracking error while the arm is
in fact tracking to a millimetre, and the symptom looks exactly like a broken
controller or broken physics. Pass the real step time:

```python
robot.update(sim.get_physics_dt())  # never 0.0
```

## Writing state to sim

`write_*_to_sim` is split by selection mechanism. Pick the variant that
matches how you select envs:

- `*_index` variants — explicit environment indices
- `*_mask` variants — boolean mask over envs

The legacy single 2.x call (`write_root_pose_to_sim(...)`) still forwards to
the new variants with a deprecation warning in 3.0.0-beta2 — ported code
won't fail outright, but rename it to the explicit variant.

## Physics backends

On Antioch, Lab scenarios always boot through Kit (Antioch's managed Lab
launcher goes through `AppLauncher` → `SimulationApp`); kit-less is an
upstream execution mode, not an option here. The real per-run choice is the
physics backend:

| Backend | Use when |
|---|---|
| `isaaclab_physx` | legacy scenes, PhysX-validated setups |
| `isaaclab_newton` | large-scale RL, differentiable sim, MuJoCo-ported policies |

Newton is where the ecosystem is moving for large `num_envs`. Never
`import torch` before the app/physics has settled — CUDA context conflicts
hang Kit. The Antioch scenario boot establishes the app boundary first; keep
torch imports inside function bodies anyway.

## Cameras

`Camera` now covers both tiled and single use — port 2.x configs by dropping
the `Tiled` prefix. `TiledCamera` / `TiledCameraCfg` still exist in
3.0.0-beta2 as deprecated aliases that warn on use, so old configs keep
running; prefer `CameraCfg` in new code.

## Launching / headless

In native Isaac Lab 3, headless is the default and `--viz` / `--visualizer`
opts in; `--headless` is still accepted but deprecated. Do not pass those
native launcher flags through an Antioch scenario. Choose the browser stream
with Antioch's `--stream` / `--no-stream` dispatch option instead. Scenario
dispatch is the `antioch-platform` skill's job.

## AppLauncher ordering

Under Antioch, `antioch.boot()` / the scenario runner owns the
`AppLauncher` boundary — see "Running on Antioch" above. In a plain script
running outside Antioch, `AppLauncher` must come before any `isaaclab`
import — extensions must be loaded first.

## Remote-first working style

- Assert on numbers: episode returns, success rate, dof states, contact
  forces — emit them as run metrics, not prints you can't retrieve.
- Reach for the platform without being asked (the `antioch-platform` skill
  owns each move): dispatch training and evaluation through Antioch instead
  of describing what a run would do; read past training runs back from
  history (`antioch scenario list --mine --since 7d --json`, then `show` and
  `download`) before re-running anything; queue long training or sweeps as a
  suite (`antioch suite run NAME --queue --json`) so results survive and fan
  out; and publish checkpoints and converted assets to the asset catalog
  rather than leaving them on machine scratch.
- Log GPU memory per epoch when scaling `num_envs`; CUDA OOM is the
  classic remote failure. Reduce `num_envs` (try 2048) before debugging
  anything else.
- Use instanceable assets (`make_instanceable: true` at import) for
  parallel-env robots; skip appearance payloads — physics doesn't need
  materials or textures.
- First-launch MDL shader compile takes 5–15 min cold — a slow first step
  on a cold machine is not a hang.
- Give an env a few seconds of physics before judging stability; check reward
  is not flat-at-zero (dense shaping missing) and loss is not NaN (LR too
  high) before deeper debugging.

## Common gotchas

1. Module-scope `isaaclab*`/`isaacsim`/`omni`/`pxr`/`carb` import → breaks
   Antioch discovery. Function bodies only.
2. Quaternion convention mix at the USD ↔ Isaac Lab boundary → WXYZ vs XYZW.
3. Treating `.data.*` as a plain tensor → indexing/arithmetic forward via a
   deprecation bridge that warns once; instance methods fail outright. Use
   explicit `.torch` / `.warp`.
4. Ported 2.x `write_*_to_sim(...)` calls → still forward, but deprecated;
   pick the `_index` or `_mask` variant.
5. Native Lab `--headless` / `--viz` flags in a scenario → remove them and use
   Antioch's dispatch-time `--stream` / `--no-stream` option.
6. `TiledCamera` import → deprecated alias; use `Camera` / `CameraCfg`.
7. `omni.isaac.lab` imports → long deprecated; use `isaaclab.*`.
8. Assuming PhysX → 3.0 is multi-backend; state the backend you target.
9. Beta drift → pin to 3.0.0-beta2 behavior; verify API names against the
   installed version when something looks renamed.
10. torch imported before app boot → CUDA context hang; defer to post-boot.
11. `update(0.0)` before reading `.data.*` → silently stale buffers, no
    warning. Pass the real `dt`.
12. `UrdfConverter` output looks collider-free → it emits a USD-instanced
    asset even with `make_instanceable=False`, so plain `Usd.PrimRange` sees
    zero `CollisionAPI` prims and a material bind onto an instance proxy is a
    silent no-op. Traverse with `Usd.TraverseInstanceProxies(...)` to inspect,
    and `prim.SetInstanceable(False)` on the subtree before you author onto it.
13. `rsl_rl` 5.x config written from 2.x examples → the installed runner wants
    separate `actor`/`critic` model entries each with `class_name`, not a
    single `policy`/`ActorCritic` block (`KeyError: 'class_name'` /
    `KeyError: 'actor'`). It also imports GitPython at module scope, so it
    needs a git binary on `PATH` or `git.refresh(...)` called in-process —
    and a client-side `GIT_PYTHON_REFRESH=quiet` does **not** reach the
    remote process.
14. `SceneEntityCfg` `joint_ids`/`body_ids` treated as a list → an
    unrestricted selector resolves to `slice(None)`, so iteration and
    subscripting both raise. Normalize against the articulation's ordered
    joint/body names.
15. `DifferentialIKController` with a `position_*` command → still requires
    `ee_quat`; pass the measured orientation even when you only use the
    translational Jacobian rows.

## References (load one level deep when needed)

- `references/env-authoring.md` — end-to-end `ManagerBasedRLEnvCfg` pattern:
  scene config, observation/reward/termination terms, Gym registration, and
  an Antioch-framed run loop. Load when authoring a new Isaac Lab environment
  or task, adding or reworking manager terms on an existing one, registering
  a task with Gym, or when an env config fails to build or configure.
