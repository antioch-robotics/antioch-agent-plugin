# Camera (Isaac Sim 6 / Kit 110)

Adapted from NVIDIA [`isaac-camera/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-camera/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Create and configure USD cameras, render products, intrinsics, AOVs, and lens distortion models for Isaac Sim 6 sensor pipelines.

Two layers stack:

1. **`UsdGeomCamera`** — the USD camera prim (focal length, apertures, clipping, transform).
2. **`isaacsim.sensors.experimental.rtx`** — the runtime sensor wrapper (authoring class + runtime sensor class). Use this for any new code that captures frames, attaches annotators, or applies lens distortion.

The classic `isaacsim.sensors.camera.Camera` still ships, but is deprecated.
Use `isaacsim.sensors.experimental.rtx` for new sensor pipelines. Enable the
owning extension through `SimulationConfig.extensions` before startup;
installed is not enabled.

> **Migration:** see the [`Migration from isaacsim.sensors.camera.Camera`](#migration-from-isaacsimsensorscameracamera) table below, plus the official [camera migration guide](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_6_0/sensors_camera_to_experimental_rtx.html#isaacsim-sensors-camera-migration) for the full API change set.

## When to use

- Create or configure camera prims in USD stages.
- Attach RGB / depth / segmentation / bbox annotators for capture.
- Apply lens distortion (OpenCV pinhole, OpenCV fisheye, LUT).
- Convert real-world calibration (OpenCV / ROS) to Omniverse units.
- Build stereo, multi-sensor, or tiled multi-camera rigs.
- Animate cameras (keyframes, motion paths, orbit, fly-through).
- Generate synthetic data with Replicator randomization.
- Set up Metropolis cameras for MEGA scenarios.

## Fundamentals

### Unit system

OpenUSD expresses optical properties in **tenths of a scene unit**.

| Property | Units | Notes |
|---|---|---|
| Focal length | tenths of scene unit | `35` = 35 mm when scene units are cm |
| Horizontal aperture | tenths of scene unit | sensor / film width |
| Vertical aperture | tenths of scene unit | sensor / film height |
| Focus distance | scene units | perfectly-sharp distance |
| Clipping range | scene units | near / far clip planes |

On a meter stage, 25 mm is raw USD `0.25`. The experimental
`cam.camera.set_focal_lengths()` wrapper instead accepts scene units:
pass `0.025` for the same lens. Convert apertures consistently; do not mix
raw USD optical units with wrapper units.

### Coordinate system

The USD optical convention: camera looks down `-Z`; `+Y` is up, `+X` is right. Same regardless of stage up-axis. ROS camera axes are `+Z` forward / `-Y` up. The classic `Camera.get_world_pose()` defaults to `camera_axes="world"` (`+X` forward, `+Z` up) — request the matching `camera_axes` value or apply one explicit conversion before using a pose in a projection matrix; do not combine the default world pose with a projection that assumes optical `-Z`.

## Creating and capturing — RtxCamera + CameraSensor

`RtxCamera` + `CameraSensor` is the way to create a camera and capture from it in Isaac Sim 6. `RtxCamera` wraps a `UsdGeom.Camera` prim with `OmniSensorAPI` applied; `CameraSensor` builds a Replicator render product on top and exposes annotators. Both extend the `isaacsim.core.experimental.prims.XformPrim` family, so the standard pose / transform methods apply.

```python
from isaacsim.sensors.experimental.rtx import RtxCamera, CameraSensor
import isaacsim.core.experimental.utils.app as app_utils

cam = RtxCamera(
    "/World/cam",
    tick_rate=30.0,  # 0 = autotrigger
)
cam.camera.set_focal_lengths(0.024)  # 24 mm on a meter stage
cam.camera.set_apertures(horizontal_apertures=0.036, vertical_apertures=0.027)
# Example 36 x 27 mm filmback for the 4:3 image below; use the task's calibration.
cam.camera.set_clipping_ranges(0.01, 1000.0)

sensor = CameraSensor(
    cam,
    resolution=(480, 640),  # (height, width)
    annotators=["rgb", "distance_to_image_plane"],
)
app_utils.play(commit=True)
data = sensor.get_data("rgb")
```

`CameraSensor.get_data()` returns a `(data, info)` tuple and may return absent data before the first produced frame — check the tuple, timestamp, and shape, and wait for a fresh sample with a bounded deadline; a fixed number of update calls does not prove readiness.

Author the pose directly on `RtxCamera` (it's an `XformPrim`): pass `positions=` / `translations=` / `orientations=` (quaternion `wxyz`) to the constructor, or call `set_world_poses(...)` afterwards.

```python
cam.set_world_poses(positions=[[2.0, 2.0, 2.0]], orientations=[[1.0, 0.0, 0.0, 0.0]])  # wxyz
```

`TiledCameraSensor` batches many cameras into one render call; useful for multi-view RL data collection. `SingleViewDepthCameraSensor` simulates stereo depth via `OmniSensorDepthSensorSingleViewAPI`.

### Lens distortion (OpenCV fisheye / pinhole)

Set via API schemas + USD attributes at authoring time. The schema and attribute names are stable; use them through `RtxCamera`'s `schemas` and `attributes` constructor args, or `Apply` them on the camera prim directly.

```python
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

Equivalent OpenCV-pinhole schema: `OmniLensDistortionOpenCvPinholeAPI` (attributes `omni:lensdistortion:opencvPinhole:{fx,fy,cx,cy,k1,k2,p1,p2,...}`). LUT-based distortion uses `OmniLensDistortionLutAPI`.

## Animation

```python
from pxr import Usd, Gf

for frame, pos in enumerate(camera_positions):
    stage.GetPrimAtPath("/World/cam").GetAttribute("xformOp:translate").Set(Gf.Vec3d(*pos), Usd.TimeCode(frame))
```

## Replicator randomization

```python
import omni.replicator.core as rep

with rep.new_layer():
    cameras = rep.get.prims(path_pattern="/World/Camera.*")
    with rep.trigger.on_frame(num_frames=200):
        with cameras:
            rep.modify.pose(position=rep.distribution.uniform((-5, -5, 2), (5, 5, 8)), look_at="/World/Target")
```

## Sensor checker / supported configs

For asset-driven lidar / camera configs and validation helpers, see `isaacsim.sensors.experimental.rtx.SUPPORTED_LIDAR_CONFIGS` and the `sensor_checker` module bundled with the extension. The Replicator AOV annotator names match the standard registry: `rgb`, `distance_to_image_plane`, `distance_to_camera`, `normals`, `semantic_segmentation`, `instance_segmentation`, `bounding_box_2d_tight`, `bounding_box_2d_loose`, `bounding_box_3d`, `camera_params`, `pointcloud`, `occlusion`.

## Migration from `isaacsim.sensors.camera.Camera`

Full guide: [`isaacsim.sensors.camera` → `isaacsim.sensors.experimental.rtx`](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_6_0/sensors_camera_to_experimental_rtx.html#isaacsim-sensors-camera-migration).

| Old | New |
|---|---|
| `isaacsim.sensors.camera.Camera("/World/cam", resolution=(1280, 720))` | `RtxCamera("/World/cam")` + `CameraSensor(cam, resolution=(720, 1280), annotators=["rgb"])` |
| `camera.set_focal_length(f)` | `cam.camera.set_focal_lengths(f)`; wrapper values use scene units |
| `camera.set_clipping_range(0.01, 1000)` | `cam.camera.set_clipping_ranges(0.01, 1000)` |
| `camera.initialize()` then `camera.get_current_frame()` | `app_utils.play(commit=True)` then `sensor.get_data(annotator)` |
| Carb-settings distortion (`/<path>/distortionModel`) | `OmniLensDistortion*API` schemas on the camera prim |
