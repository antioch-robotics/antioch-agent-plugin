# Synthetic Data Generation — Isaac Sim 6.0.1 on Antioch

Detail reference for the `isaac-sim-6` skill.

Two SDG pipelines, both authored locally with no simulator installed and
executed on Antioch GPU machines running Isaac Sim 6.0.1 / Kit 110.1.2.
Stream an interactive run while developing the pipeline, then queue production
generation headlessly:

- **Static-scene Replicator** — randomize object/camera/light poses in a frozen scene, capture annotated frames through writers.
- **MobilityGen** — record mobile-robot trajectories (physics only), then replay them with robot-mounted sensors.

Core Isaac facts — the 6.0 namespace map, boot/step/readback ordering, the
lazy-import invariant in full — live in `SKILL.md`; dispatch, queued suites,
and records in `antioch-platform`. Nothing here assumes a local simulator install.

## Managed-run capture rule

`rep.orchestrator.run()` only submits an asynchronous start command. In a
managed Antioch scenario it can return before a writer has produced any files,
so a run can appear to pass with an empty dataset. Capture each frame with
`rep.orchestrator.step(...)`, then drain
`rep.backends.io_queue.wait_until_done()` before reading the writer output.
`rep.orchestrator.wait_until_complete()` is a standalone-workflow helper and is
not the managed-run synchronization point.

## Choose the pipeline

| Goal | Pipeline |
|---|---|
| Annotated frames of a scene (object pose/camera/light randomization, no robot motion) | Static-scene Replicator |
| Robot-mounted sensor data along driven trajectories | MobilityGen record → replay+render |
| Live navigation behavior, not a dataset | the navigation reference (`references/navigation.md`) |
| Camera/LiDAR/IMU primitive setup itself | the sensors reference (`references/sensors.md`) |

## The Antioch output contract

The machine filesystem is temporary and can disappear when the machine is
released. Writers still write to a local scratch directory, but a dataset is
durable only after it becomes a run artifact:

- Write to `tempfile.mkdtemp()` inside the scenario.
- **Combine the many small files into one archive** and keep that:
  `run.add_artifact(archive_path)` — one `tar.gz` per dataset (or per episode) is the unit.
- Publish validation numbers with `run.add_result(...)` so the saved run data
  itself proves the dataset is good. Use the stream to debug, not as the only
  validation.

```python
import tarfile
import tempfile
from pathlib import Path

import antioch


def publish_dataset(run: "antioch.ScenarioRun", output_dir: Path, name: str) -> None:
    """Archive one writer output directory and keep it as a run artifact."""
    archive = output_dir.parent / f"{name}.tar.gz"
    with tarfile.open(archive, "w:gz") as tar:
        tar.add(output_dir, arcname=name)
    run.add_artifact(archive)
```

## Scale out with a queued suite

This is how SDG gets big on Antioch — not local parallel processes. Declare one
**case per seed / scene variant / episode shard**, save the selection as a
suite in `antioch.yaml`, then queue the whole sweep as one asynchronous batch.
Antioch adds the submitted project tree to the simulation image because queue
workers do not apply development watch rules.

```yaml
suites:
  sdg:
    description: "Full SDG sweep."
    select:
      - tags: ["sdg"]
```

```bash
antioch suite run sdg --queue --json               # returns the queued suite run immediately
antioch suite show SUITE_RUN_ID --follow --json    # state frames; no outcome-based exit code
```

Queued machines drain the batch under the same per-user machine quota; each case
produces its own scenario run and dataset artifact. Keep one case's
frame budget inside a run timeout — shard by case. Suite mechanics: the
`antioch-platform` skill.

## Static-scene Replicator

Replicator stays under `omni.replicator.core` in 6.0 — it was never part of the
`omni.isaac.*` purge. Scene authoring uses the experimental stage utils; never emit `isaacsim.core.utils.stage` for new code.

### Writers

Resolve via `rep.WriterRegistry.get(name)` / `rep.writers.get(name)`.

| Writer | Module | Use case |
|---|---|---|
| `BasicWriter` | `omni.replicator.core` | RGB, bbox 2D (tight/loose), bbox 3D, semantic/instance/instance-id segmentation, depth, normals, occlusion, camera params, pointcloud |
| `KittiWriter` | `omni.replicator.core` | KITTI-format datasets |
| `CocoWriter` | `omni.replicator.core` | COCO-format datasets |
| `CosmosWriter` | `omni.replicator.core` | Cosmos video clips (PNG + MP4 per modality); needs `/app/omni.graph.scriptnode/opt_in = True` |
| `PoseWriter` | `isaacsim.replicator.writers` | 6-DoF object pose estimation |

Also: `FPSWriter` (capture-rate telemetry) and custom `Writer` subclasses —
`references/sdg-replicator.md`. Deprecated, never emit: `DOPEWriter`, `YCBVideoWriter`, `PytorchWriter`, `PytorchListener`.

### Annotators

Via `rep.annotators.get(name)` / `rep.AnnotatorRegistry.get_annotator(name)`:
`rgb`, `distance_to_image_plane` (depth), `semantic_segmentation`,
`instance_segmentation`, `bounding_box_2d_tight`, `bounding_box_2d_loose`,
`bounding_box_3d`, `camera_params`, `occlusion`, `normals`, `pointcloud`. Tag
every prim you want annotated with semantics first, or segmentation and boxes
come back empty.

### Minimal Antioch scenario

Adapted from NVIDIA's `warehouse_sdg.py`: params instead of CLI flags, scratch dir + artifact instead of an output dir, results as proof.

```python
import tempfile
from pathlib import Path

import antioch


@antioch.scenario(tags=["sdg"], cases=[antioch.case(grid={"seed": range(1000, 1004)})])
def warehouse_frames(run: antioch.ScenarioRun, seed: int = 1000, num_frames: int = 64) -> None:
    """Capture annotated warehouse frames with camera and object pose randomization."""
    import numpy as np
    import omni.replicator.core as rep
    import isaacsim.core.experimental.utils.stage as stage_utils
    from isaacsim.core.experimental.utils.semantics import add_labels
    from isaacsim.storage.native import get_assets_root_path
    from pxr import Gf, UsdGeom, UsdLux

    rng = np.random.default_rng(seed)
    rep.set_global_seed(seed)
    rep.orchestrator.set_capture_on_play(False)
    assets_root = get_assets_root_path()
    stage_utils.open_stage(assets_root + "/Isaac/Environments/Simple_Warehouse/full_warehouse.usd")
    stage = stage_utils.get_current_stage()
    dome = UsdLux.DomeLight.Define(stage, "/World/SDG/DomeLight")
    dome.GetIntensityAttr().Set(400.0)
    sun = UsdLux.DistantLight.Define(stage, "/World/SDG/Sun")
    sun.GetIntensityAttr().Set(1500.0)
    UsdGeom.Xformable(sun.GetPrim()).AddRotateXYZOp().Set(Gf.Vec3f(-50, 20, 0))

    props = []  # spawn + label everything annotators should see
    for i in range(6):
        path = f"/World/SDG/cardbox_{i}"
        stage_utils.add_reference_to_stage(usd_path=assets_root + "/Isaac/Environments/Simple_Warehouse/Props/SM_CardBoxD_04.usd", path=path)
        prim = stage.GetPrimAtPath(path)
        add_labels(prim, labels=["cardbox"], taxonomy="class")
        props.append(prim)

    camera = rep.functional.create.camera(focal_length=24.0, name="DataCam")
    render_product = rep.create.render_product(camera, (1280, 720), name="main_view")

    output_dir = Path(tempfile.mkdtemp(prefix="sdg_")) / "warehouse_frames"
    writer = rep.WriterRegistry.get("BasicWriter")
    writer.initialize(output_dir=str(output_dir), rgb=True, bounding_box_2d_tight=True, semantic_segmentation=True, distance_to_image_plane=True)
    writer.attach(render_product)
    rgb = rep.AnnotatorRegistry.get_annotator("rgb")
    rgb.attach(render_product)
    logger = antioch.Logger("sdg")

    for frame in range(num_frames):
        rep.functional.modify.pose(  # randomize the camera every frame
            camera, position_value=rng.uniform((-3, -3, 1.5), (3, 3, 3.0)).tolist(), look_at_value=props[rng.integers(0, len(props))], look_at_up_axis=(0, 0, 1)
        )
        if frame % 5 == 0:  # re-scatter objects every few frames
            for prim in props:
                rep.functional.modify.pose(prim, position_value=(rng.uniform(-15, 5), rng.uniform(-5, 10), 0))
        rep.orchestrator.step(delta_time=0.0, rt_subframes=8)  # delta_time=0.0 freezes the timeline
        rep.backends.io_queue.wait_until_done()
        if frame % 8 == 0:
            # A close, explicit data camera is reviewable even when the default
            # viewport camera is distant or black.
            logger.image("camera", np.asarray(rgb.get_data()))

    rep.backends.io_queue.wait_until_done()  # flush the backend before archiving
    writer.detach()
    render_product.destroy()

    rgb_count = len(list(output_dir.glob("**/rgb/*.png")) + list(output_dir.glob("**/rgb_*.png")))
    complete = rgb_count >= num_frames
    run.check("writer produced all RGB frames", complete, detail=f"{rgb_count}/{num_frames} RGB frames")
    if not complete:
        run.fail(f"Replicator writer produced {rgb_count} RGB frames, expected {num_frames}")
    publish_dataset(run, output_dir, f"warehouse_frames_seed{seed}")  # helper from "The Antioch output contract"
    run.add_result("sdg", {"seed": seed, "frames": rgb_count})
```

Key rules for this loop:

- `rep.orchestrator.set_capture_on_play(False)` — you step captures manually;
  `delta_time=0.0` freezes the timeline between captures (static-scene SDG);
  `rep.backends.io_queue.wait_until_done()` after each step keeps the writer
  and the archive in lockstep.
- Reproducibility = one scenario `seed` param feeding both
  `np.random.default_rng(seed)` and `rep.set_global_seed(seed)`, swept by cases — same seed + same engine version ⇒ same dataset.
- Throughput: `rt_subframes` 4–8 for RTX real-time (16–32 path tracing);
  `wait_for_render=False`, Fabric writes, DLSS Quality — `references/sdg-replicator.md`.

### Configuration pattern

Scenario params are scalars, so keep the structured config — object lists,
annotation toggles, camera bounds — in a YAML file **inside the project** (the
current project tree carries it); params/cases override the sweep dimensions:

```yaml
# configs/warehouse_sdg.yaml — read from the scenario body with yaml.safe_load
resolution: [1280, 720]
rt_subframes: 8
env_url: "/Isaac/Environments/Simple_Warehouse/full_warehouse.usd"
writer: BasicWriter
annotations: {rgb: true, bounding_box_2d_tight: true, semantic_segmentation: true, distance_to_image_plane: true}
objects:
  - {url: "/Isaac/Environments/Simple_Warehouse/Props/SM_CardBoxD_04.usd", label: cardbox, count: 6}
camera: {focal_length: 24.0, position_min: [-15, -5, 1.5], position_max: [5, 10, 4.0]}
```

### Validation checklist — run it in the scenario, report as results

1. Frame count matches `num_frames` (RGB and each annotation modality).
2. RGB non-black: mean brightness > ~30 on a sample of frames.
3. Semantic labels from your `add_labels` calls appear in segmentation maps.
4. Bounding boxes have non-zero area.
5. No NaN in depth maps.

Fail the run on any failed check — a red scenario run is the signal; a dataset
archive with no results attached is not evidence.

### Gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Empty segmentation / no boxes | prims never labeled | `add_labels(prim, labels=[...], taxonomy="class")` or `rep.functional.modify.semantics(prim, {"class": label}, mode="add")` |
| Black or stale frames | capture read before render finished | `rep.backends.io_queue.wait_until_done()` after each `step()`; `rt_subframes >= 4`; discard startup captures. `rt_subframes` is per-frame temporal sampling, not the `update_app()` RTX denoiser warm-up described in `references/rendering.md` |
| `ModuleNotFoundError: omni.isaac.*` | ported 4.5/5.x code | removed in 6.0 — follow the `isaac-sim-6` namespace map |
| Archive missing trailing frames | archived before backend flush | `rep.backends.io_queue.wait_until_done()`, detach, then tar |
| `CosmosWriter` produces nothing | script-node opt-in unset | set `/app/omni.graph.scriptnode/opt_in = True` |

## MobilityGen — record → replay+render

Two phases: **record** drives a robot through a scene with physics only (no
rendering, fast) and writes trajectory state; **replay** re-loads each recording,
attaches robot-mounted sensors, and renders the dataset.

Namespace in 6.0.1: import names from the
`isaacsim.replicator.experimental.mobility_gen` package root, not deep
`...impl.*` paths; robots and scenarios stay under
`isaacsim.replicator.mobility_gen.examples`. Extensions are not auto-loaded —
enable both from inside the scenario with `app_utils.enable_extension(...)`
plus one `antioch.application().update()` before importing them, as the
skeleton below shows. (`--enable` launcher flags do exist via
`boot(extra_args=...)`, but in-scenario enabling keeps the scenario
self-contained.)

### The two-phase pattern on Antioch

**Recommended: one scenario, both phases.** Record episodes into a scratch
dir, replay them in the same process, publish the rendered dataset as one
artifact — no inter-run handoff, and record is cheap next to render. Scale by
cases: one case = one seed/episode shard, queued as a suite
(`antioch suite run NAME --queue`).

Split into two scenarios only to re-render the same trajectories with new
sensor configs and no physics re-run: record publishes `trajectories.tar.gz`
via `run.add_artifact`; replay reads it from the project checkout (download,
then `antioch assets push` — or commit small archives) and renders.

### Record phase (physics only) — skeleton

```python
@antioch.scenario(tags=["sdg", "mobilitygen"], cases=[antioch.case(grid={"seed": range(2000, 2004)})])
def mobility_dataset(run: antioch.ScenarioRun, seed: int = 2000, episodes: int = 2, max_steps: int = 2000) -> None:
    """Record Carter trajectories, then replay them with sensors into one dataset archive."""
    import isaacsim.core.experimental.utils.app as app_utils

    app_utils.enable_extension("isaacsim.replicator.experimental.mobility_gen")
    app_utils.enable_extension("isaacsim.replicator.mobility_gen.examples")
    antioch.application().update()  # one update before importing the enabled extensions
    from isaacsim.core.simulation_manager import SimulationManager
    from isaacsim.replicator.experimental.mobility_gen import Config, MobilityGenWriter
    from isaacsim.replicator.mobility_gen.examples import CarterRobot, RandomPathFollowingScenario

    # ... open stage, OccupancyMap.from_ros_yaml, SimulationManager.setup_simulation(dt=CarterRobot.physics_dt),
    # robot = CarterRobot.build("/World/robot"), scenario = RandomPathFollowingScenario.from_robot_occupancy_map(...),
    # SimulationManager.initialize_physics() — full setup in references/sdg-mobility-gen.md
    for episode in range(episodes):  # phase 1: physics-only recording
        scenario.reset()  # per episode, or the first state_dict is empty
        writer = MobilityGenWriter(str(recording_dir), async_write=True)
        writer.write_config(Config(scenario_type="RandomPathFollowingScenario", robot_type="CarterRobot", scene_usd=scene_usd))
        # ... step SimulationManager, writer.write_state_dict_common(...) per step, writer.close()
    # phase 2 (same scenario): load_scenario each recording, enable rendering, render loop — references/sdg-mobility-gen.md
```

The complete scenario — setup, the replay render loop adapted from NVIDIA's
`replay_directory.py`, and the output layout — is in `references/sdg-mobility-gen.md`; load it before writing either phase.

### Robots and scenarios

| Robot (`...mobility_gen.examples`) | Type | Notes |
|---|---|---|
| `JetbotRobot` | wheeled differential | small, `physics_dt=0.005` |
| `CarterRobot` | wheeled differential | Nova Carter, `physics_dt=0.005` |
| `H1Robot` | humanoid policy | Unitree H1 flat-terrain RL policy |
| `SpotRobot` | quadruped policy | Spot flat-terrain RL policy |

| Scenario | Mode | Headless? |
|---|---|---|
| `RandomPathFollowingScenario` | A* path to a random free-space goal, proportional steering | yes |
| `RandomAccelerationScenario` | brownian acceleration walk | yes |
| `KeyboardTeleoperationScenario` / `GamepadTeleoperationScenario` | manual | **no — needs a UI; unusable on Antioch** |

Custom robots subclass `WheeledMobilityGenRobot` (class attributes only),
`PolicyMobilityGenRobot` (implement `build_policy()`), or override
`build()`/`write_action()` for holonomic drives — full subclassing code: `references/sdg-mobility-gen.md`.

### MobilityGen pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: isaacsim.replicator.mobility_gen` (or `.experimental...`) | extension not enabled before import | `app_utils.enable_extension(...)` for both extensions, then one app update, then import |
| Replay captures nothing / "tile cannot extend outside image" | timeline never played, or an orchestrator step ran before the loop | `SimulationManager.initialize_physics()` does not play the timeline — call `omni.timeline.get_timeline_interface().play()` + one update before rendering |
| Empty/invalid `state_dict` at episode start | scenario not reset per episode | `scenario.reset()` before every episode loop |
| Custom robot: `KeyError: 'MyRobot'` at replay | `load_scenario()` resolves `ROBOTS.get(robot_type)` | register the class (`@ROBOTS.register()`) in the same process before calling `load_scenario()` and include its module through a sync rule or the sim image |
| Custom robot renders no front-camera images | missing `front_camera_*` attrs or `front_camera=None` | set the attributes and let `build()` call `cls.build_front_camera(prim_path)` |
| physics_dt mismatch between record and replay | robot params edited between phases | recording stores robot/scenario in `config.json`; keep one robot definition in project source shared by both phases |
| Occupancy map mismatch | `map.yaml` origin/resolution don't match the USD scene | regenerate or hand-check the map against the stage; MobilityGen trusts it for goals and collision checks |
| Shutdown hangs ~120 s | timeline left playing/paused at teardown | `omni.timeline.get_timeline_interface().stop()` before the run ends |

## References (load one level deep when needed)

- `references/sdg-replicator.md` — writer init forms, annotator catalog detail,
  direct-annotator synchronized capture, `rep.functional` domain randomization,
  throughput/DLSS knobs, sibling SDG workflows. Load when building or debugging
  the static-scene pipeline.
- `references/sdg-mobility-gen.md` — complete record and replay loops, replay
  output layout and state-dict format, custom robot subclassing (wheeled /
  holonomic / policy). Load when writing either MobilityGen phase or adding a
  custom robot.
