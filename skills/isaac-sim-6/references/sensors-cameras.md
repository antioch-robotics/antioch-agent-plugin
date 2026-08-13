# RTX cameras (Isaac Sim 6.0.1, on Antioch)

Deep dive on the camera stack. All snippets obey the lazy-import invariant —
imports inside function bodies. Runs execute remotely. The browser stream is
useful while framing a camera, but verify frames by shape, range, and hashes in
the saved run data.

Two layers stack:

1. **`UsdGeom.Camera`** — the USD camera prim (focal length, apertures,
   clipping, transform).
2. **`isaacsim.sensors.experimental.rtx`** — `RtxCamera` (authoring) plus
   `CameraSensor` / `TiledCameraSensor` / `SingleViewDepthCameraSensor`
   (runtime). Use these for any new code that captures frames, attaches
   annotators, or applies lens distortion.

The legacy `isaacsim.sensors.camera.Camera` still loads as a full deprecated
implementation — prefer `experimental.rtx` for new code. Migration table at
the bottom.

## Fundamentals

### Units — tenths of a scene unit

OpenUSD expresses optical properties in **tenths of a scene unit**:

| Property | Units | Notes |
|---|---|---|
| Focal length | tenths of scene unit | `35` = 35 mm when scene units are cm |
| Horizontal / vertical aperture | tenths of scene unit | sensor / film width and height |
| Focus distance | scene units | perfectly-sharp distance |
| Clipping range | scene units | near / far clip planes |

On a meter stage, a 25 mm focal length is `0.25`.

### Coordinates

The camera looks down **-Z**; `+Y` is up, `+X` is right — regardless of the
stage up-axis. Isaac Sim quaternions are `[w, x, y, z]` (see `isaac-sim-6`).

### Resolution order (one rule)

Keep this table beside the camera code: the two APIs intentionally use
opposite argument order.

| API | Resolution tuple | Example | Image array shape |
|---|---|---|---|
| New RTX sensor stack: `CameraSensor`, `TiledCameraSensor`, `SingleViewDepthCameraSensor` | `(height, width)` | `(480, 640)` | `(480, 640, channels)` |
| Replicator `rep.create.render_product` | `(width, height)` | `(640, 480)` | annotators return `(480, 640, channels)` |

The tuple is not interchangeable: convert once at the boundary when a
Replicator render product backs a new sensor.

## `RtxCamera` + `CameraSensor`

`RtxCamera` wraps a `UsdGeom.Camera` prim with `OmniSensorAPI` applied;
`CameraSensor` builds a Replicator render product on top and exposes
annotators. Only `RtxCamera` extends the
`isaacsim.core.experimental.prims.XformPrim` family — `CameraSensor` is a
`_SensorRuntime` with no transform methods, so pose the `RtxCamera` (or the
prim), never the sensor.

```python
def make_camera() -> None:
    import isaacsim.core.experimental.utils.app as app_utils
    from isaacsim.sensors.experimental.rtx import CameraSensor, RtxCamera

    cam = RtxCamera(
        "/World/cam",
        tick_rate=30.0,  # 0 = autotrigger
    )
    cam.camera.set_focal_lengths(24.0)
    cam.camera.set_clipping_ranges(0.01, 1000.0)

    sensor = CameraSensor(cam, resolution=(480, 640), annotators=["rgb", "distance_to_image_plane"])
    app_utils.play(commit=True)
    for _ in range(10):  # sensor startup; beauty-render convergence is a separate rule
        app_utils.update_app()
    rgb, info = sensor.get_data("rgb")  # (warp array | None, info dict)
    rgb_np = rgb.numpy()  # AOV payload is a CUDA warp array — convert before numpy/artifact use
```

`CameraSensor(annotators=...)` accepts only a fixed allowlist (`rgb`, depth,
normals, segmentation, bounding boxes, `pointcloud`, `motion_vectors`, …) —
`research_search` the accepted names. Anything else (`camera_params`,
`occlusion`, …) is rejected with `ValueError`; use the raw
`rep.AnnotatorRegistry` path below for those. Segmentation annotators accept
`init_params={"colorize": True}` when attached through
`rep.AnnotatorRegistry.get_annotator`.

## `TiledCameraSensor` and `SingleViewDepthCameraSensor`

- `TiledCameraSensor` batches many cameras into one render call — the tool for
  multi-view rigs and batched RL data collection. Pass camera prim **paths**
  (`str | list[str] | objects.Camera`, regex OK) — it wraps them itself, so
  you never hand it `RtxCamera` objects — and `annotators` is a required
  argument. Every tiled camera shares the render resolution; per-camera poses
  come from the camera prims.
- `SingleViewDepthCameraSensor` simulates active-stereo depth sensors (Intel
  RealSense D455/D457/D555, Stereolabs ZED X — the shipped
  `/Isaac/Sensors/` depth-camera configs) via
  `OmniSensorDepthSensorSingleViewAPI` — use it instead of hand-compositing two
  RGB cameras when the target platform is a stereo depth camera.

## Lens distortion (OpenCV pinhole / fisheye / LUT)

Distortion is authored through API schemas + USD attributes on the camera
prim — pass them as `schemas=` and `attributes=` to `RtxCamera`, or apply the
schema to an existing prim. The old Carb-settings `//distortionModel` path is
gone in 6.0.

| Model | Schema | Attribute prefix |
|---|---|---|
| OpenCV pinhole | `OmniLensDistortionOpenCvPinholeAPI` | `omni:lensdistortion:opencvPinhole:*` |
| OpenCV fisheye | `OmniLensDistortionOpenCvFisheyeAPI` | `omni:lensdistortion:opencvFisheye:*` |
| LUT lookup | `OmniLensDistortionLutAPI` | `omni:lensdistortion:lut:*` |

```python
def make_fisheye() -> None:
    from isaacsim.sensors.experimental.rtx import RtxCamera
    from pxr import Gf

    cam = RtxCamera(
        "/World/fisheye",
        schemas=["OmniLensDistortionOpenCvFisheyeAPI"],
        attributes={
            "omni:lensdistortion:opencvFisheye:fx": 500.0,
            "omni:lensdistortion:opencvFisheye:fy": 500.0,
            "omni:lensdistortion:opencvFisheye:cx": 640.0,
            "omni:lensdistortion:opencvFisheye:cy": 360.0,
            "omni:lensdistortion:opencvFisheye:k1": 0.05,
            "omni:lensdistortion:opencvFisheye:imageSize": Gf.Vec2i(1280, 720),
        },
    )
```

Pinhole attributes follow the same pattern:
`{fx, fy, cx, cy, k1, k2, p1, p2, ...}` under `opencvPinhole`.

## OpenCV intrinsics → USD camera

Convert a real calibration onto a meter stage:

```python
def opencv_to_usd_camera(stage, path, fx, fy, cx, cy, image_w, image_h, sensor_w_m=0.036):
    """OpenCV intrinsics -> USD camera on a meter-stage."""
    from pxr import UsdGeom

    focal_length_m = fx * sensor_w_m / image_w
    cam = UsdGeom.Camera.Define(stage, path)
    # USD camera attrs are in tenths of a scene unit: x10 on a meter stage
    cam.GetFocalLengthAttr().Set(focal_length_m * 10)
    cam.GetHorizontalApertureAttr().Set(sensor_w_m * 10)
    cam.GetVerticalApertureAttr().Set(fy * sensor_w_m / fx * 10)  # preserve aspect
    cam.GetHorizontalApertureOffsetAttr().Set((cx - image_w / 2) * sensor_w_m / image_w * 10)
    cam.GetVerticalApertureOffsetAttr().Set((cy - image_h / 2) * sensor_w_m / image_w * 10)
    return cam
```

## Plain `UsdGeom.Camera` + Replicator (no RTX wrapper)

When you don't need tick-rate control, lens distortion, or tiled batching,
straight USD + `omni.replicator.core` is enough:

```python
def make_plain_camera(stage) -> None:
    import omni.replicator.core as rep
    from pxr import Gf, UsdGeom

    cam = UsdGeom.Camera.Define(stage, "/World/Cam")
    cam.GetFocalLengthAttr().Set(24.0)
    cam.GetHorizontalApertureAttr().Set(36.0)  # full-frame 35 mm equivalent
    cam.GetVerticalApertureAttr().Set(20.25)
    cam.GetClippingRangeAttr().Set(Gf.Vec2f(0.01, 1000.0))

    rp = rep.create.render_product(cam.GetPath(), (1920, 1080))
    rgb = rep.AnnotatorRegistry.get_annotator("rgb")
    depth = rep.AnnotatorRegistry.get_annotator("distance_to_camera")
    semseg = rep.AnnotatorRegistry.get_annotator("semantic_segmentation", init_params={"colorize": True})
    for ann in (rgb, depth, semseg):
        ann.attach([rp])
```

For pose authoring on a plain camera, `isaacsim.core.experimental.objects.Camera`
wraps `UsdGeom.Camera` with the standard pose/scale API
(`set_world_poses(positions=..., orientations=...)`, quaternions `[w,x,y,z]`).

## Camera domain randomization

Randomize pose (and any USD attribute) per frame with Replicator triggers.
Isolate every randomization pass in `rep.new_layer()` — triggers accumulate
across runs otherwise.

```python
def randomize_camera() -> None:
    import omni.replicator.core as rep

    with rep.new_layer():
        cameras = rep.get.prims(path_pattern="/World/Camera.*")
        with rep.trigger.on_frame(num_frames=200):
            with cameras:
                rep.modify.pose(position=rep.distribution.uniform((-5, -5, 2), (5, 5, 8)), look_at="/World/Target")
```

The same `rep.modify.attribute` mechanism drives focal length, aperture, and
distortion coefficients. Scene-level material/light randomization and dataset
writer pipelines belong to the SDG reference (`references/sdg.md`), not here.

## Migration from `isaacsim.sensors.camera.Camera`

Port `Camera(...)` to `RtxCamera` + `CameraSensor` (resolution table above),
`initialize()`/`get_current_frame()` to `play(commit=True)` + warm-up +
`get_data(annotator)`, and Carb-settings distortion (`//distortionModel`,
gone in 6.0) to the `OmniLensDistortion*API` schemas. Per-method equivalents:
`research_search` the deprecated class — its docstrings name the
replacements. Keep one resolution conversion at the render-product boundary
rather than swapping dimensions in multiple callers.
