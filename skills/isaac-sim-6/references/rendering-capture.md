# Capture and frame validation

Full code for the capture pipeline summarized in `SKILL.md`. Everything runs
inside an Antioch scenario — the scenario runner owns the app boundary, so
this code never calls `antioch.boot()` or constructs `SimulationApp`. Lazy
imports inside function bodies, always.

Lighting starting point: `DomeLight` ~400 + `DistantLight` ~1500 with ACES
(the baseline recipe in `references/rendering-lighting.md`) lands a lit frame
in the validator's healthy mean range (60–180). No lights = black frame;
1000+/3000+ at default exposure overexposes.

For the one camera/render-product resolution rule, use the table in
`references/sensors-cameras.md`; this capture reference intentionally does not
repeat tuple order.

## Camera prim

Beauty-render cameras are plain USD prims, not sensor objects. Do not use
`isaacsim.sensors.camera.Camera` (sensor simulation — see `references/sensors.md`).

```python
def define_camera(stage, path: str, eye, target, focal_length: float = 20.0):
    """USD camera prim aimed with a look-at matrix."""
    from pxr import UsdGeom

    cam = UsdGeom.Camera.Define(stage, path)
    cam.CreateFocalLengthAttr().Set(focal_length)
    xf = UsdGeom.Xformable(cam.GetPrim())
    xf.AddTransformOp().Set(look_at_matrix(eye, target))
    return cam
```

## Render product + annotators

```python
def attach_annotators(camera_path: str, resolution: tuple[int, int], kinds: list[str]):
    """One render product, one annotator per AOV kind."""
    import omni.replicator.core as rep

    render_product = rep.create.render_product(camera_path, resolution)
    annotators = {}
    for kind in kinds:
        annotator = rep.AnnotatorRegistry.get_annotator(kind)
        annotator.attach([render_product])
        annotators[kind] = annotator
    return render_product, annotators
```

Render products can return empty arrays for Gaussian-splat scenes; when a
capture comes back empty and the scene uses splats, fall back to a viewport
swapchain capture at a matched window size.

## AOV dtypes and post-processing

`get_data()` returns per-annotator types — handle each differently:

| Annotator | `get_data()` returns | Post-process |
|---|---|---|
| `rgb` | (H, W, 4) uint8 RGBA | `data[:, :, :3]`, save as PNG |
| `distance_to_camera` | (H, W) float32 meters | Normalize or save as 16-bit / EXR |
| `normals` | (H, W, 4) float32 | Map [-1, 1] to [0, 255] for preview PNGs |
| `semantic_segmentation` | dict with `data` (uint32 ids) + `info` (id→class map) | Colorize ids; keep the `info` mapping |
| `instance_id_segmentation` | dict with `data` + `info` | Colorize ids |
| `motion_vectors` | (H, W, 4) float32 | Rarely needed for stills |

```python
def read_annotators(annotators) -> dict:
    """One orchestrator step reads every attached annotator."""
    import omni.replicator.core as rep

    rep.orchestrator.step()
    out = {}
    for kind, annotator in annotators.items():
        data = annotator.get_data()
        out[kind] = data[:, :, :3] if kind == "rgb" else data
    return out
```

Segmentation annotators return dicts, not arrays — `data["data"]` is the id
image and `data["info"]["idToLabels"]` maps ids to class names. Anything beyond
inspection captures (writers, randomizers, dataset on-disk layout) is SDG —
load `references/sdg.md` for that.

## Per-shot capture loop

```python
def capture_shots(run, stage, shots: list[dict], settle_frames: int = 200) -> None:
    """One render product per camera; warm RTX history, then validate."""
    import isaacsim.core.experimental.utils.app as app_utils
    from PIL import Image

    for i, shot in enumerate(shots):
        define_camera(stage, f"/World/ShotCam{i}", shot["eye"], shot["target"])
        _, annotators = attach_annotators(f"/World/ShotCam{i}", (1920, 1080), ["rgb"])
        # update_app() settles RTX denoiser/temporal accumulation; 200 is only
        # an initial budget. Start 100–200 (300–500 for deep occlusion) and
        # stop when frame statistics are stable for several samples.
        for _ in range(settle_frames):
            app_utils.update_app()
        rgb = read_annotators(annotators)["rgb"]
        ok, reason = validate_frame(rgb)
        run.add_result(f"shot_{i}_valid", ok)
        if not ok:
            run.add_result(f"shot_{i}_reject_reason", reason)
            continue
        path = f"shot_{i:05d}.png"
        Image.fromarray(rgb).save(path)
        run.add_artifact(path)
```

Validation failures are saved on the run with `run.add_result` so the
QA signal survives even when the frame is rejected. PIL is pinned in the
frozen engine runtime; reach for `imageio` or `cv2` only when the project
itself already depends on them.

## The complete frame validator

In-run (on the array, before publishing):

```python
def validate_frame(rgb) -> tuple[bool, str]:
    if rgb.max() == 0:
        return False, "no light reaches camera — add DomeLight + DistantLight"
    if rgb.mean() > 220:
        return False, f"overexposed (mean={rgb.mean():.0f}) — reduce intensity or filmIso"
    if rgb.mean() < 10:
        return False, f"underexposed (mean={rgb.mean():.0f}) — add fill lights or raise filmIso"
    if rgb.std() < 5:
        return False, f"flat frame (std={rgb.std():.1f}) — check camera aim"
    return True, f"ok (mean={rgb.mean():.0f}, max={rgb.max()}, std={rgb.std():.0f})"
```

The `std < 5` check catches what mean/max miss: a flat gray wall, a camera
buried in geometry, or an empty viewport all have nonzero mean and max but no
scene content. These statistics are only a degeneracy screen. A task camera
also needs a content oracle before publication: semantic-mask pixel count,
projected subject bounding-box occupancy, or a known-colour pixel count. A
well-exposed background with the robot outside the frame must still fail.

On downloaded artifacts (local QA after `antioch scenario download SCENARIO_RUN_ID`), file
size is the fast pre-filter before decoding:

| PNG size | Verdict |
|---|---|
| ~82 KB | Black frame |
| 200–500 KB | Partial render or trivial scene — decode and check mean |
| 1–2 MB | Fully rendered frame |

```python
def validate_artifact(path: str) -> tuple[bool, str]:
    import os

    import numpy as np
    from PIL import Image

    size_kb = os.path.getsize(path) / 1024
    if size_kb < 100:
        return False, f"black frame ({size_kb:.0f} KB)"
    rgb = np.asarray(Image.open(path).convert("RGB"))
    return validate_frame(rgb)
```

## Look-at matrix and chase-camera offsets

USD cameras look down -Z, so the matrix negates the forward axis in row 2:

```python
def look_at_matrix(eye, target, up=None):
    from pxr import Gf

    if up is None:
        up = Gf.Vec3d(0, 0, 1)
    eye = Gf.Vec3d(*eye)
    target = Gf.Vec3d(*target)
    fwd = (target - eye).GetNormalized()
    if abs(fwd * up) > 0.99:  # degenerate: looking straight up/down
        up = Gf.Vec3d(0, 1, 0)
    right = (fwd ^ up).GetNormalized()
    cam_up = (right ^ fwd).GetNormalized()
    m = Gf.Matrix4d()
    m[0] = [right[0], right[1], right[2], 0]
    m[1] = [cam_up[0], cam_up[1], cam_up[2], 0]
    m[2] = [-fwd[0], -fwd[1], -fwd[2], 0]
    m[3] = [eye[0], eye[1], eye[2], 1]
    return m
```

Third-person offsets (Z-up, robot facing +X at yaw = 0):

| Direction | Vector |
|---|---|
| Behind robot | `-X` |
| Right of robot | `-Y` |
| Left of robot | `+Y` |
| Above robot | `+Z` |

```python
def chase_position(robot_xy, yaw: float, behind: float, side: float, height: float):
    """Camera XY behind and beside a robot at `yaw`; flip `side` to change sides."""
    import math

    x = robot_xy[0] - behind * math.cos(yaw) - side * math.sin(yaw)
    y = robot_xy[1] - behind * math.sin(yaw) + side * math.cos(yaw)
    return (x, y, height)
```

`side = -2.5` puts the camera on the robot's right, `+2.5` on the left. Flip
the offset value, never the trig signs.

## Dynamic chase-camera height

In cluttered scenes a fixed-height chase camera clips into tall geometry.
Pre-compute obstacle bounds once, then raise the camera per frame:

```python
def collect_obstacles(stage) -> list[tuple[float, float, float, float, float]]:
    """(xmin, xmax, ymin, ymax, height) per obstacle prim, once at startup."""
    from pxr import UsdGeom

    cache = UsdGeom.BBoxCache(0, ["default"])
    obstacles = []
    for prim in stage.Traverse():
        if not prim.IsA(UsdGeom.Gprim):
            continue
        bound = cache.ComputeWorldBound(prim).ComputeAlignedBox()
        lo, hi = bound.GetMin(), bound.GetMax()
        obstacles.append((lo[0], hi[0], lo[1], hi[1], hi[2]))
    return obstacles


def clearance_height(obstacles, cx: float, cy: float, margin: float = 0.5) -> float:
    """Highest obstacle near (cx, cy); the camera must clear this."""
    highest = 0.0
    for xmn, xmx, ymn, ymx, h in obstacles:
        if xmn - margin <= cx <= xmx + margin and ymn - margin <= cy <= ymx + margin:
            highest = max(highest, h)
    return highest


def smooth_height(current: float, target: float) -> float:
    """Per-frame exponential smoothing toward the required height."""
    return current * 0.95 + target * 0.05
```

## Video assembly

```bash
ffmpeg -y -framerate 30 -i frames/frame_%05d.png \
    -c:v libx264 -pix_fmt yuv420p -crf 18 output.mp4
```

- Numbering must be gapless — ffmpeg stops silently at the first gap.
- Use one zero-padded global counter across episodes, or one directory per
  episode and assemble per-episode clips.
- `-crf 18` is visually lossless for review; raise to 23 for smaller files.
- Assembling on the machine before `run.add_artifact("output.mp4")` ships one
  artifact instead of thousands of PNGs; assembling locally after download
  keeps the raw frames for re-encoding.
