# Isaac Sim Headless Rendering (Kit 110 / Isaac Sim 6.0+)

Adapted from NVIDIA [`isaac-sim-rendering/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/SKILL.md) (Apache-2.0).

[Isaac Sim task index](../SKILL.md#domain-references) · [Antioch startup](../../antioch-platform/references/simulation-code.md)

Capture pipeline, lighting recipes, ACES calibration, camera math, validation.

The lighting recipes below are examples from upstream warehouse scenes. Their values are starting points, not universal settings or measurements reproduced on Antioch. Warm-up depends on assets, shaders, temporal accumulation, and renderer — confirm readiness with frame-content checks and a per-frame deadline; a fixed settle-frame count never proves readiness, and a dark or bright frame can be intentional.

## Upstream source examples

| Script | Purpose | Arguments |
|---|---|---|
| [`scripts/capture_pipeline.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/scripts/capture_pipeline.py) | Standard Kit 110 / Isaac Sim 6.0+ headless capture pipeline | see script --help |
| [`scripts/look_at_camera.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/scripts/look_at_camera.py) | Look-at camera math for USD cameras (Z-up, USD -Z forward convention) | see script --help |
| [`scripts/warehouse_lighting.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/scripts/warehouse_lighting.py) | Multi-layer warehouse lighting recipes for headless Isaac Sim rendering | see script --help |

## Read first

- [navigation-primitives](navigation-primitives.md): look-at chase camera math (cross-referenced).

## Capture: Replicator RGB annotator

Standard Kit 110 / Isaac Sim 6.0+ capture pipeline. Works headless, including on ARM64.

`setup_capture_pipeline(stage_path, width, height, renderer, settle_frames)` — open stage, define camera, attach RGB annotator, settle, return `(app, rgb_annot, render_product)`. `capture_frame(rgb_annot)` — step replicator and return `(H, W, 3)` uint8 RGB array.

See [`scripts/capture_pipeline.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/scripts/capture_pipeline.py).

For live controller demos where simulation remains the time authority, disable Replicator capture-on-play and capture snapshots without pausing the timeline. See [`scripts/capture_pipeline.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/scripts/capture_pipeline.py) for the executable pattern.

In a live Jupyter kernel, use `await rep.orchestrator.step_async(...)`.
Do not start a nested event loop or synchronous Kit update loop while an
asynchronous capture is running. A standalone script owns its own loop.

`SimulationConfig.viewport_updates` controls the default viewport: `None` follows `stream`, `True` keeps it rendering, and `False` pauses it, including the stream. Sensor and Replicator render products have their own updates. Keep pumping Kit when those products need it; pausing the default viewport is not permission to stop the application loop.

**Capture method choice**:
- `omni.replicator.core` RGB annotator -> reliable, supports any resolution.
- `RtxCamera` + `CameraSensor` (from `isaacsim.sensors.experimental.rtx`) for tick-rate control, OpenCV / fisheye lens distortion, ISP, tiled multi-view, or stereo depth (see [isaac-camera](isaac-camera.md)).
- Swapchain capture -> also works on Kit 110 if you explicitly set window size matching the render resolution.
- Replicator render products may return empty arrays for Gaussian splat scenes; fall back to swapchain capture in that case.

## Choose and tune the renderer

Use RTX real-time for iterative work when it meets the task. Path tracing
can be useful for offline fidelity comparisons; profile its convergence and
cost on the actual scene. Startup and frame timings depend on assets,
shaders, resolution, and hardware, not one fixed settle count.

```python
import carb

settings = carb.settings.get_settings()
settings.set("/rtx/rendermode", "RayTracedLighting")
# For an offline path-traced comparison:
# settings.set("/rtx/rendermode", "PathTracing")
```

| Mode | Useful for | Check |
|---|---|---|
| RTX real-time | Iteration, live cameras, trajectory capture | Temporal artifacts, shadows, noise, frame cost |
| Path tracing | Offline comparisons and converged images | Samples, noise, lighting convergence, total render cost |

Run these fragments in the initialized remote process. Choose the renderer for the requested output and measure the scene's actual cost. There is no fixed number of seconds or frames that establishes convergence.

## Headless Lighting — Add Explicit Lights

Do not rely on a desktop viewport's default lighting. Inspect the scene's
authored lights and emissive materials. This dome-plus-sun recipe is one
starting point, not a required pair:

```python
from pxr import UsdLux, UsdGeom, Gf

dome = UsdLux.DomeLight.Define(stage, "/World/DomeLight")
dome.GetIntensityAttr().Set(400.0)

sun = UsdLux.DistantLight.Define(stage, "/World/Sun")
sun.GetIntensityAttr().Set(1500.0)
UsdGeom.Xformable(sun.GetPrim()).AddRotateXYZOp().Set(Gf.Vec3f(-50, 20, 0))
```

Tune lighting, exposure, and tone mapping together against a fresh image.
The upstream [warehouse lighting example](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/scripts/warehouse_lighting.py)
shows one arrangement; its intensities are scene-specific. Inspect deep
occlusion and bright open views separately rather than adding uniform fill
until a brightness threshold passes.

## Exposure and tone mapping

Tune exposure with lighting rather than increasing every light until the image becomes visible. The upstream recipe uses ACES; the shipped Kit settings enumerate ACES as operator **6**, not the upstream example's 4. The white point setting is an RGB color, not a Kelvin temperature.

```python
import carb

settings = carb.settings.get_settings()
settings.set("/rtx/post/tonemap/op", 6)  # ACES at this runtime pin
settings.set("/rtx/post/tonemap/filmIso", 200.0)
settings.set("/rtx/post/tonemap/whitepoint", (1.0, 1.0, 1.0))
```

Film ISO, exposure time, f-number, automatic exposure, and the camera pipeline can all affect the result. Inspect their current settings before changing them. For controlled comparisons, keep exposure and tone mapping fixed across images. A camera's ISP configuration may need separate treatment; see [camera calibration and capture](isaac-camera.md).

The upstream warehouse experiments used these film ISO starting points. Hold the rest of the camera and lighting configuration fixed when comparing them:

| View | Upstream film ISO |
|---|---|
| General warehouse | 200 |
| Deep aisle | 600 |
| Aerial or overview | 400 |

## Warehouse lighting example

The upstream [warehouse lighting helper](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/scripts/warehouse_lighting.py) combines ambient light, ceiling fixtures, and optional atmosphere. Adapt its setup to the authored scene instead of treating every layer as mandatory.

| View | Starting adjustment | Compare against |
|---|---|---|
| Open overview | Low ambient fill plus directional or ceiling light | Bright floors, shadow detail, clipped highlights |
| Deep aisle | Fixtures that actually illuminate the aisle and its lower shelves | Ground-level robot cameras and occluded surfaces |
| Object close-up | Local key and fill with enough separation from the background | Material appearance, reflections, fine geometry |

One upstream recipe paired **film ISO 600 and no dome light** with an 8 × 14 grid of ceiling panels: intensity 70,000, size 2.5 × 1.5 m, and warm color `(1.0, 0.97, 0.92)`. Aisle sphere lights used intensity 15,000, radius 0.1 m, and height 3.5 m. Upstream reported mean RGB values of 60–155 across its views; these measurements have not been reproduced on Antioch. Treat this as a separate recipe from the dome-plus-sun example above, and retune it for the actual scene.

```python
from pxr import Gf, UsdGeom, UsdLux

panel = UsdLux.RectLight.Define(stage, "/World/Lighting/CeilingPanel")
panel.GetWidthAttr().Set(2.5)
panel.GetHeightAttr().Set(1.5)
panel.GetIntensityAttr().Set(70000.0)
panel.GetColorAttr().Set(Gf.Vec3f(1.0, 0.97, 0.92))
UsdGeom.Xformable(panel.GetPrim()).AddTranslateOp().Set(Gf.Vec3d(0, 0, 8))
```

This example assumes a Z-up stage in meters and a new owned light path. A rect light faces local -Z, so this panel points down. Place fixtures from the real aisle layout. Do not brighten an entire warehouse to fix one occluded camera: compare both the deep aisle and open overview after each change. Fog can help a requested visual appearance but also changes sensor imagery; do not add it to an evaluation without making it an input.

### Deep aisles and unsuccessful recipes

In the upstream 3.5 m wide, 8 m tall aisle, ceiling lights at 10 m did not adequately illuminate the floor. Lower aisle lights and a camera at a cross-aisle junction gave a clearer view. Its experiments also found that very wide rect lights flattened the light pools, a dome at intensity 400 or above washed out shadows at film ISO 600, and Reinhard tone mapping looked muddy in that scene. Those observations are useful comparisons, not renderer-wide rules.

The upstream helper used fog density 0.003 for a 40 m hall. Start there only when that atmosphere suits the task, then inspect the result. Fixed warm-up counts and reported path-tracing times also depend on the scene and hardware; use fresh frame checks and measured cost rather than treating them as guarantees.

## Frame quality validation

Decode and inspect the actual frame before delivery. Check dimensions, channels,
freshness, camera framing, lighting, materials, and the requested appearance.
Compressed file size cannot distinguish a valid simple scene from a black frame.

Pixel statistics help diagnosis but do not define a universal quality gate:

```python
import numpy as np

pixels = np.asarray(rgb_array)
if pixels.size == 0:
    raise RuntimeError("no image data")
print({"shape": pixels.shape, "min": pixels.min(), "max": pixels.max(), "mean": pixels.mean()})
```

Unexpected all-zero pixels can mean no light, wrong framing, or an unready
sensor. Very bright or dark images may be intentional; compare with the task.

## Look-At Camera Math

For a target-facing camera, a look-at matrix avoids manual Euler-angle tuning.

`look_at_matrix(eye, target, up)` — returns `Gf.Matrix4d` for a USD camera at `eye` looking at `target`. Handles degenerate up-vector (straight down/up).

See [`scripts/look_at_camera.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/scripts/look_at_camera.py).

### Third-Person Camera Offsets (Z-up, robot facing +X at yaw=0)

| Direction | Vector |
|---|---|
| Behind robot | `-X` |
| Right of robot | `-Y` |
| Left of robot | `+Y` |
| Above robot | `+Z` |

```python
import math

behind_dir_x = -math.cos(yaw)
behind_dir_y = -math.sin(yaw)
left_dir_x = -math.sin(yaw)
left_dir_y = math.cos(yaw)

cam_x = robot_x + behind_dist * behind_dir_x + side_offset * left_dir_x
cam_y = robot_y + behind_dist * behind_dir_y + side_offset * left_dir_y
cam_z = height
```

- `side_offset = -2.5` → camera on robot's **right**
- `side_offset = +2.5` → camera on robot's **left**
- Flip the *offset value* to change sides, NOT the trig signs.

## Dynamic Camera Height (Obstacle Avoidance)

When tracking through cluttered environments, the chase camera will clip into tall geometry. Pre-compute obstacle bboxes, then raise the camera each frame as needed.

```python
# Build obstacle lookup from USD geometry once at startup
obstacles = []
for prim in stage.Traverse():
    if prim.IsA(UsdGeom.Cube):
        # ... extract (xmin, xmax, ymin, ymax, height) ...
        obstacles.append((xmin, xmax, ymin, ymax, height))


def cam_max_height_at(cx, cy, margin=0.5):
    """Highest obstacle near (cx, cy). Camera must clear this."""
    return max((h for xmn, xmx, ymn, ymx, h in obstacles if xmn - margin <= cx <= xmx + margin and ymn - margin <= cy <= ymx + margin), default=0.0)


# Per-frame:
target_h = max(base_height, cam_max_height_at(cam_x, cam_y) + 1.0)
smooth_h = smooth_h * 0.95 + target_h * 0.05  # smooth transitions
```

## Preserve the robot's transforms

For initial placement, prefer a parent wrapper over clearing the referenced
robot's transform stack. During physics, use the articulation/controller
API; direct root transforms are appropriate for explicitly labeled replay.

## Video Assembly

```bash
ffmpeg -y -framerate 30 -i frames/frame_%05d.png \
  -c:v libx264 -pix_fmt yuv420p -crf 18 output.mp4
```

Keep frame numbering sequential (`frame_00000.png`, `frame_00001.png`, …). The image-sequence reader can stop at a gap; it does not preserve missing time for you. Use capture timestamps to choose playback rate, or a timestamp-aware workflow when cadence varies.

## Session Management

For batch/iterative rendering, keep the session's Kit app running and switch stages in-place rather than restarting:

- Reuse the initialized app to avoid repeated startup; latency depends on scene and renderer
- Use `success, stage = stage_utils.open_stage(path)` (`isaacsim.core.experimental.utils.stage`) to switch scenes; check `success` before using the stage
- Restart the exact kernel for a new Kit lifecycle; image/dependency changes need a new session
- Recreate stage-bound handles after opening another stage

Use an interactive kernel or a long-running scenario for this (see [agentic-simulation](../../agentic-simulation/SKILL.md)) — the principle is "don't pay the cold-start cost more than once."

## Checklist before delivering renders

1. Renderer, lighting, exposure, and lens match the intended appearance.
2. The frame is fresh and the scene has converged within a bounded deadline.
3. The decoded image shows the requested subject from the intended viewpoint.
4. Capture did not change physics or sensor configuration unintentionally.
5. Video frames have ordered timestamps and complete task coverage.

Continue with [data collection](data-collection-sim.md) for annotated datasets, [viewport capture](../../agentic-simulation/references/viewport.md) for quick inspection, or [scenario telemetry](../../scenario-design/references/telemetry.md) to keep rendered evidence with a run.
