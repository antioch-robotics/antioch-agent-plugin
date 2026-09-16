# Isaac Sim Headless Rendering (Kit 110 / Isaac Sim 6.0+)

Adapted from NVIDIA [`isaac-sim-rendering/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Capture production-quality headless frames with RT2 or PathTracing, ACES tone mapping, warehouse lighting patterns, and quantitative validation thresholds.

Capture pipeline, lighting recipes, ACES calibration, camera math, validation.

All light intensities, filmIso values, and settle-frame counts below are scene-specific tuning starting points validated on upstream warehouse scenes, not universal values. Warm-up depends on assets, shaders, temporal accumulation, and renderer — confirm readiness with frame-content checks and a per-frame deadline; a fixed settle-frame count never proves readiness, and a dark or bright frame can be intentional.

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

**Capture method choice**:
- `omni.replicator.core` RGB annotator -> reliable, supports any resolution.
- `RtxCamera` + `CameraSensor` (from `isaacsim.sensors.experimental.rtx`) for tick-rate control, OpenCV / fisheye lens distortion, ISP, tiled multi-view, or stereo depth (see [isaac-camera](isaac-camera.md)).
- Swapchain capture -> also works on Kit 110 if you explicitly set window size matching the render resolution.
- Replicator render products may return empty arrays for Gaussian splat scenes; fall back to swapchain capture in that case.

## RT2 vs PathTracing

```python
settings.set("/rtx/rendermode", "RayTracedLighting")  # RT2 — real-time
# settings.set("/rtx/rendermode", "PathTracing")      # offline only
```

| Mode | Convergence | Per-frame time | Use for |
|---|---|---|---|
| **RayTracedLighting (RT2)** | ~200 settle frames (~10-15s) | 10-15s | All iterative work, warehouse scenes, training data |
| **PathTracing** | converges over many subframes | 5-30 min | Final hero shots only, when explicitly requested |

**Default to RT2.** Switch to PathTracing only after RT2 has been calibrated and the user asks for hero quality.

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

### Baseline Intensity Guide

| Scene Type | DomeLight | DistantLight | Notes |
|---|---|---|---|
| Warehouse (default) | 400 | 1500 | Good general balance |
| Close-up robot | 300 | 1200 | Slightly softer |
| Outdoor | 500 | 2000 | Brighter sun |
| Dark/moody | 100 | 800 | Dramatic shadows |

## ACES Tone Mapping — The Single Biggest Quality Lever

ACES is one useful tone-mapping choice for indoor scenes. Tune exposure and
lighting together against the requested appearance.

```python
import carb

s = carb.settings.get_settings()

s.set("/rtx/post/tonemap/op", 4)  # ACES
s.set("/rtx/post/tonemap/filmIso", 600.0)  # key parameter (see table)
s.set("/rtx/post/tonemap/whitepoint", 6500.0)
s.set("/rtx/post/tonemap/enabled", True)
s.set("/rtx/post/aa/op", 3)  # TAA for RT2
```

### filmIso Calibration (validated on warehouse interiors)

| Scene | filmIso | Notes |
|---|---|---|
| General warehouse RT2 | 200 | Photorealistic starting point |
| Deep-aisle indoor (hero camera) | 600 | Best balance across hero/overview/aisle/topdown |
| Aerial/overview-heavy | 400 | Avoid overexposure on open views |

### Anti-Recipes (don't waste time on these)
- Wide rect lights (width=5+) → flat, no light pools
- High dome intensity (400+) with ACES filmIso 600 → washes out shadows
- Reinhard tonemapping → muddy, low contrast
- PathTracing for iterative work → 5-30 min per frame, kills velocity

## Warehouse lighting example

`add_warehouse_lighting(stage, n_lights, settings)` — low-ambient dome + focused rect lights + optional fog. Pass `settings=carb.settings.get_settings()` to enable fog.

See [`scripts/warehouse_lighting.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-rendering/scripts/warehouse_lighting.py).

For 40m warehouse: fog density 0.003 adds depth without murk.

## Deep-Aisle Indoor Lighting

### Problem
Ground-level camera in narrow aisle = black frame (82KB / mean_RGB < 5). Ceiling rect lights at Z=10m can't illuminate a 3.5m-wide × 8m-tall aisle to ground level — RT2 struggles with deep occlusion.

### Solution: Multi-Layer Lighting
```python
# Layer 1: dense ceiling grid (6×12 across facility)
# Rect lights at ceil_z-0.3, pointing down
# intensity=200000, width=4.0, height=3.0  (wide coverage)

# Layer 2: low sphere lights IN each aisle at Z=3.5m (head-height)
# Directly in camera FOV between tier 1 and ground
for aisle_y, lx in aisle_light_positions:
    lt = UsdLux.SphereLight.Define(stage, lp)
    lt.GetRadiusAttr().Set(0.15)
    lt.GetIntensityAttr().Set(100000.0)
```

- The upstream example used 500 settle frames; measure readiness and convergence on the current scene.
- Dome at 300 intensity is optional ambient fill — don't go higher or open views wash out

### Dome vs Deep-Aisle Tension (fundamental conflict in enclosed scenes)
- High dome → overview/topdown overexpose (mean > 220)
- Low/no dome → deep aisle underexpose (mean < 10)
- **Best balance**: no dome + sphere lights in aisles + 500K rect grids + 500 settle frames
  - Hero aisle: mean ~60
  - Overview (elevated 3/4): mean ~140-175
  - Cross-aisle: mean ~230

### Validated ACES filmIso=600 Light Intensities
- Ceiling rect lights: **70,000** intensity, 2.5×1.5m, warm white (1.0, 0.97, 0.92)
- Aisle sphere lights: **15,000** intensity, radius=0.1, at Z=3.5m
- Grid: 8×14 ceiling panels
- **No dome light** — ACES handles exposure
- Result: mean 60–155 across all view types

**Camera tip**: place "hero" camera at cross-aisle intersections, not deep in narrow aisles. The junction has more open space for light to reach.

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

For chase/POV/overview cameras pointing at a target, always use a look-at matrix. Don't hand-tune Euler angles — they're brittle and you'll waste hours on sign flips.

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
right_dir_x = -math.sin(yaw)
right_dir_y = math.cos(yaw)

cam_x = robot_x + behind_dist * behind_dir_x + side_offset * right_dir_x
cam_y = robot_y + behind_dist * behind_dir_y + side_offset * right_dir_y
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

## Robot XformOp Discipline

URDF-imported robots (Spot, Carter, etc.) already have authored `translate + orient + scale` xformOps on the root prim.

- Use `xf.ClearXformOpOrder(); xf.MakeMatrixXform()` on the **root prim only** for initial placement.
- **Never** add ops to child body/link prims — physics drives those.

## Video Assembly

```bash
ffmpeg -y -framerate 30 -i frames/frame_%05d.png \
  -c:v libx264 -pix_fmt yuv420p -crf 18 output.mp4
```

Frame numbering must be **sequential** (`frame_0000.png`, `frame_0001.png`, …) — ffmpeg skips gaps.

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

## Integration Points

- **RECEIVES from:** [urdf-mjcf-to-usd-conversion](urdf-mjcf-to-usd-conversion.md), [usd-articulation](usd-articulation.md), [mobility-gen](mobility-gen.md), [isaac-sim-robot-navigation](isaac-sim-robot-navigation.md) — populated stages to render
- **PRODUCES for:** [data-collection-sim](data-collection-sim.md) — validated frame sequences for SDG
- **PRODUCES for:** [isaac-sim-validator](isaac-sim-validator.md) — outputs for final QA gate
