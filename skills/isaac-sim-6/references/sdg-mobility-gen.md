# MobilityGen record → replay+render — depth

Load this when writing either MobilityGen phase or adding a custom robot. Runs
on Antioch against Isaac Sim 6.0.1 / Kit 110.1.2: core API from
`isaacsim.replicator.experimental.mobility_gen`, robots/scenarios from
`isaacsim.replicator.mobility_gen.examples`, all imports inside the scenario
body, output published as one archived artifact.

## The complete one-scenario pipeline

Record episodes with physics only, replay each with sensors, publish the
rendered dataset. One case per seed; queue the sweep as a suite
(`antioch suite run NAME --queue`). Antioch adds the submitted project source
to the simulation image because queue workers do not apply development watch
rules.

```python
import asyncio
import tempfile
from pathlib import Path

import antioch


@antioch.scenario(tags=["sdg", "mobilitygen"], cases=[antioch.case(grid={"seed": range(2000, 2004)})])
def mobility_dataset(run: antioch.ScenarioRun, seed: int = 2000, episodes: int = 2, max_steps: int = 2000, render_interval: int = 40) -> None:
    """Record Carter trajectories, then replay them with sensors into one archived dataset."""
    import numpy as np
    import isaacsim.core.experimental.utils.app as app_utils

    app_utils.enable_extension("isaacsim.replicator.experimental.mobility_gen")
    app_utils.enable_extension("isaacsim.replicator.mobility_gen.examples")
    antioch.application().update()

    import omni.replicator.core as rep
    import omni.timeline
    from isaacsim.core.experimental.objects import GroundPlane
    from isaacsim.core.experimental.utils.stage import get_current_stage, open_stage
    from isaacsim.core.simulation_manager import SimulationManager
    from isaacsim.replicator.experimental.mobility_gen import (
        Config,
        MobilityGenReader,
        MobilityGenWriter,
        OccupancyMap,
        apply_sensor_overrides,
        collect_input,
        load_scenario,
        save_sensor_overrides,
    )
    from isaacsim.replicator.mobility_gen.examples import CarterRobot, RandomPathFollowingScenario
    from isaacsim.storage.native import get_assets_root_path
    from pxr import UsdGeom

    np.random.seed(seed)
    rep.set_global_seed(seed)
    scratch = Path(tempfile.mkdtemp(prefix="mobilitygen_"))
    scene_usd = get_assets_root_path() + "/Isaac/Environments/Simple_Warehouse/warehouse_multiple_shelves.usd"
    omap_yaml = Path("configs/warehouse_map/map.yaml")  # available through a sync rule or the built sim image
    open_stage(scene_usd)
    for _ in range(30):  # let async USD/material loading finish before physics
        antioch.application().update()
    occupancy_map = OccupancyMap.from_ros_yaml(str(omap_yaml))

    SimulationManager.setup_simulation(dt=CarterRobot.physics_dt)
    ground_plane = GroundPlane("/World/ground_plane", templates=None)  # invisible safety floor
    for mesh_path in ground_plane.meshes.paths:
        UsdGeom.Imageable(get_current_stage().GetPrimAtPath(mesh_path)).MakeInvisible()
    robot = CarterRobot.build("/World/robot")
    scenario = RandomPathFollowingScenario.from_robot_occupancy_map(robot, occupancy_map)
    SimulationManager.initialize_physics()
    antioch.application().update()
    recordings = scratch / "recordings"
    config = Config(scenario_type="RandomPathFollowingScenario", robot_type="CarterRobot", scene_usd=scene_usd)
    # Collect a self-contained scene copy once; each recording embeds it via
    # copy_stage so load_scenario can open the recording's own stage.usd.
    stage_cache = Path(tempfile.mkdtemp(prefix="mobilitygen_stage_"))
    cached_stage_path = asyncio.get_event_loop().run_until_complete(collect_input(scene_usd, str(stage_cache)))
    for episode in range(episodes):  # ---- phase 1: record (physics only)
        scenario.reset()
        recording = recordings / f"episode_{episode:04d}"
        writer = MobilityGenWriter(str(recording), async_write=True)
        writer.write_config(config)
        writer.write_occupancy_map(occupancy_map)
        writer.copy_stage(cached_stage_path)
        save_sensor_overrides(robot.prim_path, str(recording))
        for step in range(max_steps):
            SimulationManager.step(steps=1)
            antioch.application().update()
            if not scenario.step(CarterRobot.physics_dt):  # goal reached or collision
                break
            writer.write_state_dict_common(scenario.state_dict_common(), step=step)
        writer.close()
    replays = scratch / "replays"
    timeline = omni.timeline.get_timeline_interface()
    frames_rendered = 0
    for recording in sorted(recordings.iterdir()):  # ---- phase 2: replay + render
        loaded = load_scenario(str(recording))
        SimulationManager.initialize_physics()
        loaded.enable_rgb_rendering()
        loaded.enable_segmentation_rendering()
        loaded.enable_depth_rendering()  # enable_normals_rendering() is also available
        loaded.finalize_rendering()
        timeline.play()  # captureOnPlay gates capture on a playing timeline; do NOT pre-step the orchestrator
        antioch.application().update()
        apply_sensor_overrides("/World/robot", str(recording))
        reader = MobilityGenReader(str(recording))
        writer = MobilityGenWriter(str(replays / recording.name))
        if len(reader) > 0:  # warm the RTX temporal accumulator; these frames are discarded
            warmup_state = reader.read_state_dict(index=0)
            for _ in range(4):
                loaded.load_state_dict(warmup_state)
                loaded.write_replay_data()
                SimulationManager.step(steps=1)
                antioch.application().update()
                rep.orchestrator.step(rt_subframes=1, delta_time=0.0, pause_timeline=False)
        for step in range(0, len(reader), render_interval):
            state = reader.read_state_dict(index=step)
            loaded.load_state_dict(state)
            loaded.write_replay_data()
            SimulationManager.step(steps=1)  # sync PhysX tensor writes back to USD before rendering
            antioch.application().update()
            rep.orchestrator.step(rt_subframes=1, delta_time=0.0, pause_timeline=False)
            loaded.update_state()
            merged = loaded.state_dict_common()
            for key, value in state.items():  # physics truth stays the recorded values
                if value is not None:
                    merged[key] = value
            writer.write_state_dict_common(merged, step)
            writer.write_state_dict_rgb(loaded.state_dict_rgb(), step)
            writer.write_state_dict_segmentation(loaded.state_dict_segmentation(), step)
            writer.write_state_dict_depth(loaded.state_dict_depth(), step)
            frames_rendered += 1
        rep.backends.io_queue.wait_until_done()
        timeline.stop()  # leaving the timeline playing across teardown crashes Kit
        loaded.disable_rendering()
        writer.close()

    assert frames_rendered > 0, "replay rendered no frames"
    publish_dataset(run, replays, f"mobility_dataset_seed{seed}")  # helper from references/sdg.md ("The Antioch output contract")
    run.add_result("mobilitygen", {"seed": seed, "episodes": episodes, "frames": frames_rendered})
```

Lineage: phase 1 follows NVIDIA's `benchmark_mobility_gen_recording.py`, phase 2
condenses `replay_directory.py`; both adapted to the Antioch run model.
`render_interval` is physics steps per rendered frame — 40 at `physics_dt=0.005`
(200 Hz physics) yields ~5 Hz of frames.

## Output layout

```text
recordings/<episode>/
  config.json              # {"scenario_type", "robot_type", "scene_usd"}
  stage.usd                # self-contained scene copy (collect_input + copy_stage)
  sensor_overrides.usda    # save_sensor_overrides sidecar, re-applied at replay
  occupancy_map/           # map.yaml + map.png
  state/common/<step>.npz  # poses, joint positions/velocities — one numpy dict per step
replays/<episode>/         # plus state/common merged with the recorded truth
  state/rgb/<camera>/<step>.jpg
  state/segmentation/<camera>/<step>.png
  state/depth/<camera>/<step>.png  # 16-bit inverse depth
  state/normals/<camera>/<step>.npy
```

`load_scenario()` opens the recording's own copied stage (`stage.usdz`, or
`stage.usd` after stripping stale SDGPipeline prims), so a recording is
self-contained and portable across machines. Archive the
`replays/` tree; keep `recordings/` only to re-render with different sensors
(the split two-scenario handoff in references/sdg.md, "The two-phase
pattern on Antioch").

## Custom robots

Define the class in a project module (e.g. `src/robots.py`) and import it
**inside the scenario body** — the class body executes simulator imports, so
the module must never load during local discovery. Registration lets
`load_scenario()` resolve `config.json`'s `robot_type` in both phases.

### Wheeled (differential drive) — attributes only

`WheeledMobilityGenRobot.build()` / `write_action()` are inherited; a subclass
is a bag of class attributes:

```python
# src/robots.py — module-scope sim imports are safe here only because this
# module is never on the scenario discovery path; import it inside the body.
from isaacsim.replicator.experimental.mobility_gen import ROBOTS
from isaacsim.replicator.mobility_gen.examples import WheeledMobilityGenRobot
from isaacsim.replicator.mobility_gen.examples.misc import HawkCamera


@ROBOTS.register()
class MyRobot(WheeledMobilityGenRobot):
    physics_dt: float = 0.005
    z_offset: float = 0.25
    chase_camera_base_path = "chassis"  # required by load_scenario even headless
    chase_camera_x_offset: float = -1.5
    chase_camera_z_offset: float = 0.8
    chase_camera_tilt_angle: float = 60.0
    front_camera_base_path = "chassis/front_hawk"  # without these, replay renders nothing
    front_camera_rotation = (0.0, 0.0, 0.0)
    front_camera_translation = (0.2, 0.0, 0.1)
    front_camera_type = HawkCamera
    occupancy_map_radius: float = 0.5  # footprint the planner/collision checks assume
    occupancy_map_z_min: float = 0.1
    occupancy_map_z_max: float = 0.5
    occupancy_map_cell_size: float = 0.05
    occupancy_map_collision_radius: float = 0.5
    random_action_linear_velocity_range = (-0.3, 1.0)  # keyboard_/gamepad_*_gain attrs exist too, teleop-only
    random_action_angular_velocity_range = (-0.75, 0.75)
    random_action_linear_acceleration_std: float = 5.0
    random_action_angular_acceleration_std: float = 5.0
    random_action_grid_pose_sampler_grid_size: float = 5.0
    path_following_speed: float = 1.0
    path_following_angular_gain: float = 1.0
    path_following_stop_distance_threshold: float = 0.5
    path_following_forward_angle_threshold = 0.785
    path_following_target_point_offset_meters: float = 1.0
    wheel_dof_names = ["left_wheel_joint", "right_wheel_joint"]
    usd_url: str = "/path/to/my_robot.usd"
    chassis_subpath: str = "chassis"
    wheel_base: float = 0.5
    wheel_radius: float = 0.1
```

References to copy from: `JetbotRobot` (`wheel_base=0.1125`, `wheel_radius=0.03`)
and `CarterRobot` (`wheel_base=0.413`, `wheel_radius=0.14`, `chassis_subpath="chassis_link"`).

### Holonomic (e.g. Kaya) — override `build()` and `write_action()`

```python
from isaacsim.storage.native import get_assets_root_path
import numpy as np


@ROBOTS.register()
class KayaRobot(WheeledMobilityGenRobot):
    # ... same attribute blocks as the wheeled example, plus:
    wheel_dof_names = ["axle_0_joint", "axle_1_joint", "axle_2_joint"]
    usd_url: str = get_assets_root_path() + "/Isaac/Robots/NVIDIA/Kaya/kaya.usd"
    chassis_subpath: str = "base_link"
    wheel_radius: float = 0.04
    wheel_base: float = 0.1
    com_prim_subpath: str = "base_link/control_offset"

    @classmethod
    def build(cls, prim_path: str) -> "KayaRobot":
        from isaacsim.core.experimental.prims import Articulation
        from isaacsim.core.experimental.utils.stage import add_reference_to_stage, get_current_stage
        from isaacsim.robot.experimental.wheeled_robots import HolonomicController
        from isaacsim.robot.experimental.wheeled_robots.robots.holonomic_robot_usd_setup import HolonomicRobotUsdSetup

        add_reference_to_stage(usd_path=cls.usd_url, path=prim_path)
        get_current_stage(backend="usd").Load(prim_path)
        setup = HolonomicRobotUsdSetup(robot_prim_path=prim_path, com_prim_path=f"{prim_path}/{cls.com_prim_subpath}")
        wheel_radius, wheel_positions, wheel_orientations, mecanum_angles, wheel_axis, up_axis = setup.get_holonomic_controller_params()
        controller = HolonomicController(
            wheel_radius=wheel_radius,
            wheel_positions=wheel_positions,
            wheel_orientations=wheel_orientations,
            mecanum_angles=mecanum_angles,
            wheel_axis=wheel_axis,
            up_axis=up_axis,
        )
        return cls(prim_path=prim_path, articulation=Articulation(prim_path), controller=controller, front_camera=cls.build_front_camera(prim_path))

    def write_action(self, step_size: float) -> None:
        if not self.is_physics_ready():
            return
        if self._wheel_indices is None:
            self._wheel_indices = self.articulation.get_joint_indices(self.wheel_dof_names)
        linear, angular = self.action.get_value()  # MobilityGen's 2D action
        wheel_velocities = self.controller.forward(command=[linear, 0.0, angular])  # [forward, lateral, yaw]
        self.articulation.set_dof_velocities(wheel_velocities[np.newaxis], dof_indices=self._wheel_indices)
```

### Policy-based (legged) — implement `build_policy()`

`write_action()` already maps `[linear, angular]` to the 3D command
`[x, 0, yaw]`; references: `H1Robot` (`articulation_path="pelvis"`),
`SpotRobot` (`articulation_path="/"`):

```python
@ROBOTS.register()
class MyLeggedRobot(PolicyMobilityGenRobot):
    physics_dt: float = 0.005
    z_offset: float = 1.05
    controller_z_offset: float = 1.05  # policy spawn height; H1Robot/SpotRobot keep it separate from MobilityGen's z_offset
    usd_url: str = "/path/to/robot.usd"
    articulation_path = "pelvis"
    # ... same occupancy_map_*, random_action_*, path_following_*, camera attrs

    @classmethod
    def build_policy(cls, prim_path: str) -> "H1FlatTerrainPolicy":
        import numpy as np
        from isaacsim.robot.policy.examples.robots import H1FlatTerrainPolicy

        return H1FlatTerrainPolicy(prim_path=prim_path, position=np.array([0.0, 0.0, cls.controller_z_offset]))
```

## Pitfalls beyond "MobilityGen pitfalls" in references/sdg.md

- **Do not mix MobilityGen's flow with your own `World`.** The runner owns
  boot; MobilityGen drives `SimulationManager` directly. Never call legacy
  `World.reset()` / `initialize_simulation_context()` around it — the
  initialize call is async-only and raises `AttributeError`.
- **`physics_dt` is recorded, not negotiated.** `config.json` pins
  scenario/robot/scene; replay uses `robot_type.physics_dt`. Editing robot
  attrs between phases corrupts the dataset silently. Keep one robot
  definition in the project and include it through the same watch rule or sim
  image in both phases.
- **The occupancy map is an input you must ship.** Origin and resolution in
  `map.yaml` must match the USD scene or goals spawn inside shelves. Generating
  the map is the navigation stack's job (`references/navigation.md`) — commit the
  `map.yaml` + PNG pair into the project so Antioch includes both files.
- **One GPU, one process.** Multi-GPU replay segfaults on Kit 110.1.x; one
  scenario process per Antioch machine is already the safe shape.
