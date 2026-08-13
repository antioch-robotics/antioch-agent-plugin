# Sensors — Isaac Sim 6.0.1 on Antioch

Detail reference for the `isaac-sim-6` skill.

Sensor code here is authored locally with no simulator installed and executes
inside an Antioch scenario process on a remote GPU machine running Isaac Sim
6.0.1 / Kit 110.1.2. Stream the GUI to the browser while debugging, but verify
sensor frames in the scenario process and save them as run metrics
(`run.add_result`), artifacts (`run.add_artifact`), or Rerun telemetry
(`antioch.Logger`).

Shared Isaac facts — the lazy-import invariant, the 6.0 namespace map,
boot/step/readback ordering, `play(commit=True)` before `get_data()`, the
XformCache trap, quaternion conventions — live in this skill's `SKILL.md`;
load it first if you have not. Dispatch (`antioch scenario run` / `antioch run`)
and record inspection live in the `antioch-platform` skill. This skill
covers only what is sensor-specific.

## Rule #1, sensor edition

The lazy-import invariant (SKILL.md) bites sensors twice: sensor class
imports go inside the scenario function body, and a Replicator `Writer`
subclass (the GMO pattern below) inherits from `omni.replicator.core.Writer`,
so the **class definition itself** must live inside a function — a
module-scope `class MyWriter(Writer)` is a module-scope `omni` import.

## Choosing the sensor stack

| You need | Authoring class | Runtime class | Module |
|---|---|---|---|
| Camera RGB / depth / AOVs | `RtxCamera` | `CameraSensor` | `isaacsim.sensors.experimental.rtx` |
| Many cameras, batched render | camera prim paths | `TiledCameraSensor` | same |
| Stereo depth (RealSense / ZED style) | `RtxCamera` | `SingleViewDepthCameraSensor` | same |
| Vendor lidar (Ouster, HESAI, SICK, …) | `Lidar.create(config=…)` | `LidarSensor` | same |
| Custom-scan lidar | USDA prim + `Lidar(path)` | `LidarSensor` | same |
| Radar | `Radar` | `RadarSensor` | same |
| Ultrasonic / acoustic | `Acoustic` | `AcousticSensor` | same |
| Contact / force | `Contact.create()` | `ContactSensor` | `isaacsim.sensors.experimental.physics` |
| IMU | `IMU.create()` | `IMUSensor` | same |
| Joint effort / joint state | (joint already exists) | `EffortSensor` / `JointStateSensor` | same |

The legacy `isaacsim.sensors.camera.Camera` and `isaacsim.sensors.physics.*`
classes still load in 6.0, but their implementations moved to the
`experimental.*` extensions — deprecated; prefer the `experimental.*` stack
for new code. Antioch blocks construction of the legacy camera because its
second render product can hang `World.step(render=True)` forever on the stock
Isaac Sim 6.0.1 image. For a screenshot of the active USD camera, use the
bounded `antioch.capture_viewport()` helper instead:

```python
def capture_active_view() -> object | None:
    # Point `/OmniverseKit_Persp` (or another active USD camera) with Kit's
    # normal viewport helpers before this call.
    import antioch

    return antioch.capture_viewport()
```

The helper reads Antioch's existing viewport product and never creates a
second product. It returns `None` when the active viewport has no product yet;
retry after a rendered step or use `isaacsim.sensors.experimental.rtx` for
per-camera RGB/depth/AOV data.

## The two readback models — pick the right one

| Sensor family | Readback | Why |
|---|---|---|
| Camera AOVs (`CameraSensor`) | `sensor.get_data(name)` after play + warm-up steps | AOVs are per-render-product, tick-aligned |
| Lidar / radar / acoustic GMO | `sensor.attach_writer(...)` with a `Writer` subclass | GMO frames emit asynchronously under multitick — polling `get_data("generic-model-output")` drops or duplicates frames |
| Physics sensors | `sensor.get_data()` after `play(commit=True)` | Synchronous per-step readings |

## RTX camera quick path

```python
def capture_overhead(run: antioch.ScenarioRun) -> None:
    import isaacsim.core.experimental.utils.app as app_utils
    from isaacsim.sensors.experimental.rtx import CameraSensor, RtxCamera

    cam = RtxCamera("/World/overhead", tick_rate=30.0)
    cam.camera.set_focal_lengths(24.0)
    cam.camera.set_clipping_ranges(0.01, 1000.0)
    sensor = CameraSensor(
        cam,
        resolution=(720, 1280),  # see sensors-cameras.md for tuple order
        annotators=["rgb", "distance_to_image_plane"],
    )
    app_utils.play(commit=True)
    for _ in range(10):  # sensor startup only; beauty RTX warm-up uses frame-stat convergence
        app_utils.update_app()
    rgb, info = sensor.get_data("rgb")  # (warp array | None, info dict)
    rgb_np = rgb.numpy()  # AOV payload is a CUDA warp array — convert before numpy/artifact use
```

The `CameraSensor` annotator name list, distortion, intrinsics conversion,
tiled/stereo rigs, and camera randomization: `references/sensors-cameras.md`.

## Lidar / radar / acoustic — the GMO Writer pattern

RTX non-camera sensors emit a `GenericModelOutput` (GMO) buffer through the
sensor scheduler, zero or many frames per step. The Replicator scheduler calls
`Writer.write` on every produced output, so a writer sees every event with no
gaps. Note the writer class is defined **inside** the function (Rule #1):

```python
def collect_lidar_frames() -> list:
    import isaacsim.core.experimental.utils.app as app_utils
    import omni.replicator.core as rep
    from isaacsim.sensors.experimental.rtx import Lidar, LidarSensor, parse_generic_model_output_data
    from omni.replicator.core import Writer

    frames = []

    class GmoCollectWriter(Writer):
        def __init__(self) -> None:
            self.data_structure = "renderProduct"
            self.annotators = [rep.annotators.get("GenericModelOutput")]

        def write(self, data) -> None:
            if "renderProducts" not in data:
                return
            for _rp, rp_data in data["renderProducts"].items():
                raw = rp_data.get("GenericModelOutput")
                if isinstance(raw, dict):
                    raw = raw.get("data")
                gmo = parse_generic_model_output_data(raw)
                if gmo.numElements > 0:
                    frames.append(gmo)

    rep.WriterRegistry.register(GmoCollectWriter)

    lidar = Lidar.create(
        path="/World/lidar",
        config="OS1",  # stem alias for a SUPPORTED_LIDAR_CONFIGS asset path
        variant="OS1_REV6_32ch20hz512res",
        aux_output_level="FULL",
        tick_rate=20.0,
        accumulate_outputs=True,
    )
    sensor = LidarSensor(lidar, annotators=[])  # the writer brings its own
    sensor.attach_writer("GmoCollectWriter")

    app_utils.play(commit=True)
    for _ in range(120):
        app_utils.update_app()
    return frames
```

`Radar` / `RadarSensor` and `Acoustic` / `AcousticSensor` consume GMO through
the identical pattern. Vendor catalog, `Lidar.create` argument semantics,
mount attachment, custom scan patterns, GMO field layouts, and radar/acoustic
specifics: `references/sensors-rtx.md`.

## Physics sensors (contact, IMU, effort, joint state)

Synchronous per-step readings; require `play(commit=True)` before `get_data()`.
`EffortSensor` and `JointStateSensor` are runtime-only wrappers around an
existing joint prim — no authoring class.

```python
def read_physics_sensors() -> None:
    import isaacsim.core.experimental.utils.app as app_utils
    from isaacsim.sensors.experimental.physics import Contact, ContactSensor, EffortSensor, IMU, IMUSensor, JointStateSensor

    imu = IMUSensor(IMU.create(path="/World/Robot/imu"))
    # imu.get_data() frame keys: linear_acceleration, angular_velocity, orientation
    contact = ContactSensor(
        Contact.create(
            path="/World/Robot/foot/contact",
            min_threshold=0.0,
            max_threshold=1e6,
            radius=-1,  # stub-proven: negative disables radius filtering
        )
    )
    effort = EffortSensor("/World/Robot/joint_arm_1")
    joint = JointStateSensor("/World/Robot/joint_arm_1")

    app_utils.play(commit=True)
    app_utils.update_app()
    imu_frame = imu.get_data()
    contact_frame = contact.get_data()
```

Deeper physics-sensor context (thresholds, raycast queries) lives in the
`isaac-sim-6` skill's physics reference.

## Feeding results back to the run

Sensor reads are only useful if they are saved with the run. Emit scalars as
metrics, arrays/images as artifacts, time series as telemetry:

```python
def report(run: antioch.ScenarioRun, points, rgb) -> None:
    import numpy as np

    ranges = np.linalg.norm(points, axis=1)
    run.add_result(
        "sensors",
        {"lidar_hit_count": int(points.shape[0]), "lidar_min_range_m": round(float(ranges.min()), 4), "lidar_max_range_m": round(float(ranges.max()), 4)},
    )
    np.save("/tmp/lidar_points.npy", points)  # host numpy — warp arrays need .numpy() first
    run.add_artifact("/tmp/lidar_points.npy", content_type="application/octet-stream")
```

- Define the fail-loud signal **before** running: hit counts, range bounds,
  frame shapes, dynamic range, and content hashes. Assert on them in the
  scenario so an empty or stale sensor cannot look successful.
- Poll with a deadline, not a fixed count, when warming up a render product or
  GMO stream; assert when the deadline expires so a dead sensor fails loudly.
- A module-scope `antioch.Logger("sensors")` streams time series and point
  clouds into the run's `.rrd` telemetry through `.scalar(...)` / `.value(...)`
  — the remote stand-in for a GUI. Never construct `antioch.Logger()` per
  sample inside a step loop: it is a constructor in a hot loop, and its empty
  prefix silently writes to a different channel.

## `aux_output_level` by modality

Per-modality levels — including why cameras have no such parameter — live in
`references/sensors-rtx.md`.

## Gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Discovery fails locally with `ModuleNotFoundError: pxr`/`omni` | Module-scope simulator import — often a module-scope `class W(Writer)` | Move imports and the writer class inside the function |
| Camera AOV is `None` / empty right after play | Render product has produced no frame yet | `app_utils.update_app()` in a warm-up loop with a deadline before `get_data()` |
| Lidar frames dropped or duplicated | GMO emits asynchronously under multitick; polling per step mismatches | Use `attach_writer`, not per-step `get_data("generic-model-output")` |
| Physics sensors return empty/stale | Timeline not playing | `app_utils.play(commit=True)` before any `get_data()` |
| Camera resolution looks transposed | Sensor and render-product APIs use different tuple orders | Use the one resolution table in `references/sensors-cameras.md` |
| `usdrt.population.plugin Unhandled attribute type VtArray` log spam | Expected when `aux_output_level` is set | Harmless — the Replicator pipeline still picks the attribute up |
| Annotator data missing in writer | Annotators attached after orchestrator start | Attach writers/annotators before `play()` |

RTX lidar/radar specifics (variant selection, radar motion BVH):
`references/sensors-rtx.md`.

## References (load one level deep when needed)

- `references/sensors-cameras.md` — `RtxCamera`/`CameraSensor` detail, `TiledCameraSensor`,
  `SingleViewDepthCameraSensor`, lens distortion (OpenCV pinhole/fisheye/LUT),
  OpenCV-intrinsics conversion, camera domain randomization, units and
  coordinates, legacy-`Camera` migration. Load when configuring camera
  intrinsics, distortion, multi-camera rigs, or camera randomization.
- `references/sensors-rtx.md` — `SUPPORTED_LIDAR_CONFIGS` vendor catalog,
  `Lidar.create` variants, USDA mount attachment, custom emitter-state scan
  patterns, GMO decode helpers and per-modality field layouts, radar (motion
  BVH) and acoustic specifics. Load when working with vendor lidar configs,
  custom scan patterns, GMO parsing, radar, or acoustic sensors.
