# MobilityGen Synthetic Data Generation

Adapted from NVIDIA [`mobility-gen/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Run MobilityGen two-phase SDG: record robot trajectories headlessly, then replay and render RGB/depth/segmentation/normal/pose outputs.

Two-phase pipeline: **record trajectories** (physics, no rendering) → **replay & render** (sensors added).

## Upstream source examples

| Script | Purpose | Arguments |
|---|---|---|
| [`scripts/custom_footprint_robot.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/scripts/custom_footprint_robot.py) | Custom robot footprint factory for MobilityGen SDG | see script --help |
| [`scripts/holonomic_robot_subclass.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/scripts/holonomic_robot_subclass.py) | Example holonomic (3-wheel) MobilityGenRobot subclass (Kaya) | see script --help |
| [`scripts/record_trajectories.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/scripts/record_trajectories.py) | Phase 1 trajectory recording for MobilityGen SDG | see script --help |
| [`scripts/replay_custom_robot.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/scripts/replay_custom_robot.py) | Replay recordings with a custom robot registered at runtime | see script --help |
| [`scripts/wheeled_robot_subclass.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/scripts/wheeled_robot_subclass.py) | Example WheeledMobilityGenRobot subclass for custom differential-drive robots | see script --help |

## Read These Skills First

- **[navigation-primitives](navigation-primitives.md)** — `OccupancyMap`, A* planner, robot footprints (Spot Z=0.69), differential/holonomic kinematics, look-at chase cameras, shared gotchas. MobilityGen consumes this substrate; this skill assumes you know it.
- **[occupancy-map](occupancy-map.md)** — produces the `map.yaml` consumed by `OccupancyMap.from_ros_yaml`
- **[data-collection-sim](data-collection-sim.md)** — sibling SDG path for static scenes with randomized object/camera poses (no robot trajectory)

## When To Use This Skill (vs siblings)

| Goal | Use |
|---|---|
| Record trajectories then re-render with sensors for SDG (training data) | **this skill** |
| Drive a robot through a scene in real time, see it move | [isaac-sim-robot-navigation](isaac-sim-robot-navigation.md) |
| Annotated frames with no robot motion (object pose randomization) | [data-collection-sim](data-collection-sim.md) |

## Related Skills

- [navigation-primitives](navigation-primitives.md) — shared navigation substrate (read first)
- [data-collection-sim](data-collection-sim.md) — static-scene SDG sibling
- [isaac-sim-sensor](isaac-sim-sensor.md) — sensor primitives (camera, LiDAR, IMU, contact)
- [isaac-sim-robot-navigation](isaac-sim-robot-navigation.md) — runtime navigation sibling
- [antioch-platform](../../antioch-platform/SKILL.md) — headless startup, session dispatch, and stream configuration

## Environment

There is no local Isaac install or `python.sh`; scripts run in the Antioch simulator service.

- **Replay script**: upstream source example [`source/standalone_examples/replicator/mobility_gen/replay_directory.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/standalone_examples/replicator/mobility_gen/replay_directory.py).
- **Extension examples**: upstream [`source/extensions/isaacsim.replicator.mobility_gen.examples/`](https://github.com/isaac-sim/IsaacSim/tree/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/source/extensions/isaacsim.replicator.mobility_gen.examples).
- **Data dir**: `$MOBILITY_GEN_DATA` (env var). Point it at a project-local path such as `./MobilityGenData`.
  - `recordings/` — timestamped trajectory dirs
  - `replays/` — rendered output
  - `maps/` — occupancy map YAML + PNG files

## Extension Loading (Critical)

Extensions are **not auto-loaded**. Upstream passes `--enable` flags to `python.sh`; in Antioch declare the extension IDs in `SimulationConfig.extensions` instead:

```python
# scenario decorator or plain-script startup config
extensions = ["isaacsim.replicator.mobility_gen.examples"]
# add "isaacsim.asset.gen.omap" when generating occupancy maps
```

Enabling `isaacsim.replicator.mobility_gen.examples` auto-loads `isaacsim.replicator.experimental.mobility_gen` as a dependency. All extension-dependent imports must come AFTER simulation startup (`antioch.start_simulation()` or the scenario's managed start).

Import public names from the package root:

```python
from isaacsim.replicator.experimental.mobility_gen import ROBOTS, SCENARIOS, OccupancyMap, RecordingSession, load_scenario
```

`OccupancyMapDataValue` is not re-exported; import it from `...mobility_gen.impl.occupancy_map`.

## Phase 1: Automated Trajectory Recording (Headless)

`KeyboardTeleoperationScenario` and `GamepadTeleoperationScenario` require an interactive UI. For headless batch recording use `RandomPathFollowingScenario` or `RandomAccelerationScenario`.

> **API note (Kit 110):** MobilityGen no longer uses the legacy `World` flow.
> `get_world()` / `new_world()` are gone, and so is `impl.utils.global_utils`.
> Recording is driven by `RecordingSession` plus the
> `isaacsim.core.simulation_manager.SimulationManager` lifecycle, with
> `isaacsim.core.experimental.utils.stage` for stage I/O (note `open_stage()`
> returns a `(bool, stage)` tuple, and `save_stage()` takes only a path).
>
> **Migration:** for the full `omni.isaac.*` → `isaacsim.*` mapping when porting scripts off the legacy World flow, see [Renaming Extensions](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_4_5/extensions_renaming.html).

`record_trajectories(scene_usd, omap_yaml, robot_type, scenario, num_episodes, max_steps, data_dir)` — headless SimulationApp loop that builds a robot and scenario and records each episode to `$MOBILITY_GEN_DATA/recordings/`.

`RecordingSession` call order — the session owns the ground plane, robot spawn, `Config` and writer, so scripts do not construct a `MobilityGenWriter` themselves:

```python
session = RecordingSession()
session.build(robot_cls, scenario_cls, occupancy_map, scene_usd=..., cached_stage_path=..., recordings_dir=...)
omni.timeline.get_timeline_interface().play()  # initialize() expects a playing app
simulation_app.update()
session.initialize()
session.reset()
session.enable_recording()
while ...:
    SimulationManager.step(steps=1)  # initialize_physics() does not start the
    simulation_app.update()  # timeline, so update() alone won't tick physics
    if not session.step(robot_cls.physics_dt):
        break
```

See [`scripts/record_trajectories.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/scripts/record_trajectories.py).

## Phase 2: Replay & Render

Replay all recordings in `$MOBILITY_GEN_DATA/recordings/` and write sensor data to `replays/`.

Run the upstream `replay_directory.py` pattern as a plain script in the Antioch service (or fold its loop into a scenario), with `isaacsim.replicator.mobility_gen.examples` declared in `SimulationConfig.extensions`:

```bash
export MOBILITY_GEN_DATA=./MobilityGenData
python replay_directory.py \
  --input  "$MOBILITY_GEN_DATA/recordings" \
  --output "$MOBILITY_GEN_DATA/replays" \
  --render_interval 40 \
  --rgb_enabled True \
  --depth_enabled True \
  --segmentation_enabled True \
  --normals_enabled False \
  --render_rt_subframes 1
```

`--render_interval 40` = 1 frame per 40 physics steps (~5 Hz at 200 Hz physics). Increase `--render_rt_subframes` for better quality at the cost of speed.

### Replay Output Structure

```
replays/<recording_name>/
  config.json
  stage.usd
  occupancy_map/map.yaml, map.png
  state/
    common/<step>.npy          # robot pose, joint positions, velocities
    rgb/<camera_name>/<step>.jpg
    segmentation/<camera_name>/<step>.png
    depth/<camera_name>/<step>.png   # 16-bit inverse depth
    normals/<camera_name>/<step>.npy
```

## Available Robots

| Name | Type | Notes |
|---|---|---|
| `JetbotRobot` | Wheeled (differential) | Small, `physics_dt=0.005`, Jetbot USD |
| `CarterRobot` | Wheeled (differential) | Nova Carter, `physics_dt=0.005` |
| `H1Robot` | Humanoid (policy) | Unitree H1, flat-terrain RL policy |
| `SpotRobot` | Quadruped (policy) | Boston Dynamics Spot, flat-terrain RL policy |
| `CarterMultiSensorRobot` | Wheeled, sensor rig | Rig loaded from `data/robots/carter.yaml` |
| `JetbotMultiSensorRobot` | Wheeled, sensor rig | Rig loaded from `data/robots/jetbot.yaml` |
| `H1MultiSensorRobot` | Humanoid, sensor rig | Rig loaded from `data/robots/h1.yaml` |
| `SpotMultiSensorRobot` | Quadruped, sensor rig | Rig loaded from `data/robots/spot.yaml` |

The four `*MultiSensorRobot` variants subclass `MobilityGenMultiSensorRobot` and
declare their cameras in a YAML sensor-rig config instead of the
`front_camera_*` class attributes used by the single-camera robots above.

## Available Scenarios

| Name | Mode | Headless? |
|---|---|---|
| `KeyboardTeleoperationScenario` | Manual (WASD) | No — needs UI |
| `GamepadTeleoperationScenario` | Manual (gamepad) | No — needs UI |
| `RandomAccelerationScenario` | Automated (brownian) | Yes |
| `RandomPathFollowingScenario` | Automated (A* path following) | Yes |

`RandomPathFollowingScenario` plans an A* path from the robot's current position to a random free-space goal and follows it with proportional steering. Episode ends when goal is reached or robot collides.

## Add a Custom Robot

Two base classes exist depending on robot type. Both handle `build()` and `write_action()` — set class-level attributes only.

### Wheeled (differential drive)

Subclass `WheeledMobilityGenRobot`. No need to override `build()` or `write_action()`:

`MyRobot(WheeledMobilityGenRobot)` — example class showing all required class-level attributes (camera offsets, occupancy-map params, velocity ranges, wheel geometry) with no method overrides needed.

See [`scripts/wheeled_robot_subclass.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/scripts/wheeled_robot_subclass.py).

Reference implementations in `isaacsim.replicator.mobility_gen.examples.robots`:
- `JetbotRobot`: NVIDIA Jetbot, `wheel_base=0.1125`, `wheel_radius=0.03`, `chassis_subpath="chassis"`
- `CarterRobot`: Nova Carter, `wheel_base=0.413`, `wheel_radius=0.14`, `chassis_subpath="chassis_link"`

### Holonomic (e.g. Kaya 3-wheel)

Override `build()` to use a different controller and `write_action()` to remap the 2D action:

`KayaRobot(WheeledMobilityGenRobot)` — overrides `build()` to configure a `HolonomicController` from `HolonomicRobotUsdSetup`, and `write_action()` to map `[lin, ang]` to `[forward, lateral=0, yaw]`.

See [`scripts/holonomic_robot_subclass.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/scripts/holonomic_robot_subclass.py).

### Policy-based (legged robots)

Subclass `PolicyMobilityGenRobot` and implement `build_policy()`. `write_action()`
converts the 2D action `[lin_vel, ang_vel]` into `[x, 0, yaw]`. Use the pinned
policy class; newer upstream `RobotPolicyRunner` / `get_*_spec` examples do not
match this runtime. This H1-specific example mirrors the shipped `H1Robot`:

Class attributes are the same `occupancy_map_*`, `random_action_*`, and `path_following_*` set as
the wheeled robot, plus `articulation_path` and `controller_z_offset`.

```python
import numpy as np
from isaacsim.replicator.experimental.mobility_gen import ROBOTS
from isaacsim.replicator.mobility_gen.examples.robots import PolicyMobilityGenRobot
from isaacsim.robot.policy.examples.robots import H1FlatTerrainPolicy


@ROBOTS.register()
class MyLeggedRobot(PolicyMobilityGenRobot):
    physics_dt: float = 0.005
    z_offset: float = 1.05
    articulation_path = "pelvis"
    controller_z_offset: float = 1.05

    @classmethod
    def build_policy(cls, prim_path: str) -> H1FlatTerrainPolicy:
        return H1FlatTerrainPolicy(prim_path=prim_path, position=np.array([0.0, 0.0, cls.controller_z_offset]))
```

Reference implementations: `H1Robot` (`articulation_path="pelvis"`) and `SpotRobot` (`articulation_path="/"`) in the same module.

### Replay with a custom robot

`replay_directory.py` calls `load_scenario()` which does `ROBOTS.get(config.robot_type)`. If the robot isn't in the built-in extension, this raises `KeyError`. **You cannot pass `--enable` to load an ad-hoc Python file** — either create a proper Isaac extension, or copy the replay loop into your own script and register the robot class before calling `load_scenario()`:

`replay_with_custom_robot(input_dir, custom_robot_class)` — register a custom robot class at runtime, then call `load_scenario()` for each recording directory.

See [`scripts/replay_custom_robot.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/mobility-gen/scripts/replay_custom_robot.py).

## Config / Data Format

`config.json` per recording:
```json
{
  "scenario_type": "RandomPathFollowingScenario",
  "robot_type": "CarterRobot",
  "scene_usd": "/path/to/warehouse.usd"
}
```

`state/common/<step>.npy` is a numpy dict: `position`, `orientation`, `joint_positions`, `joint_velocities`, `linear_velocity`, `angular_velocity`.

## Common Pitfalls

- **`ModuleNotFoundError: No module named 'isaacsim.replicator.experimental.mobility_gen'`**: Extensions aren't auto-loaded. Enable `isaacsim.replicator.mobility_gen.examples` via `SimulationConfig.extensions`. All extension imports must come AFTER simulation startup.
- **`KeyError: 'CarterRobot'` from `ROBOTS.get(...)` despite a clean import**: `ROBOTS` came from a different registry than the one the examples extension populates. Import it from `isaacsim.replicator.experimental.mobility_gen`.
- **`ImportError: ...impl.utils.global_utils`** or **`get_world` / `new_world` / `join_sdf_paths` undefined**: removed with the legacy `World` flow. Use `RecordingSession` + `SimulationManager`, and `isaacsim.core.experimental.utils.prim.join_prim_paths`.
- **Recording runs but every episode has 0 steps**: `session.step()` was called without advancing physics. `SimulationManager.initialize_physics()` does not start the Kit timeline, so `simulation_app.update()` alone does not tick physics — call `SimulationManager.step(steps=1)` each iteration.
- **Replay `KeyError: 'MyRobot'`**: `replay_directory.py` only knows built-in robots. Write a wrapper script that registers your robot class before calling `load_scenario()`.
- **Custom robot produces no images during replay**: Missing `front_camera_*` attributes, or `build()` passes `front_camera=None`. Add the attributes and call `cls.build_front_camera(prim_path)` in `build()`.
- **`AttributeError: 'MyRobot' has no attribute 'chase_camera_base_path'`**: `chase_camera_base_path`, `chase_camera_x_offset`, `chase_camera_z_offset`, `chase_camera_tilt_angle` are required by `load_scenario()` even for headless recording.
- **Built-in replay fails to find robot class**: Enable `isaacsim.replicator.mobility_gen.examples` via `SimulationConfig.extensions` so the examples extension registers its robots/scenarios before `load_scenario()` runs.
- **`physics_dt` mismatch**: Recording stores the physics timestep in `config.json`; replay uses the same `robot_type.physics_dt`. Do not change robot params between record and replay.
- **Occupancy map scale**: MobilityGen consumes the `OccupancyMap` produced by [occupancy-map](occupancy-map.md). Ensure `map.yaml` origin and resolution match the USD world coordinates.
