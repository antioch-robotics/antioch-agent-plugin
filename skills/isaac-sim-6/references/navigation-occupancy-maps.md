# Occupancy Maps from USD

Two programmatic paths to a 2D occupancy grid, plus ROS export. Both run inside
the scenario process. Use the browser stream to inspect the live scene, then
save the grid or a preview as durable run evidence. All simulator imports stay
inside function bodies (Rule #1, `isaac-sim-6`).

## Reality check: direct projection is a first-class path

The live pinned image raised `ModuleNotFoundError` when the omap binding was
imported before its extension was enabled. Direct USD projection is therefore
the reliable, extension-free path: it works from authored bounds and does not
depend on the optional omap extension. Use the Generator when you specifically
need PhysX collision raycasts and can verify the extension in the run.

## Path 1: direct USD projection (no extension)

Rasterize authored world AABBs onto a grid. Use when the scene lacks
colliders, when you need a deterministic projection from authored geometry,
or when prototyping with placeholder shapes. It does not respect physics
collision approximations.

### Collider-driven traversal (preferred)

Iterating only prims with `CollisionAPI` excludes visual-only geometry
(signage, decals, light cones, debug arrows) without a filter list:

```python
def rasterize_obstacles(grid: "np.ndarray", x_range: tuple, y_range: tuple, resolution: float) -> None:
    import numpy as np
    from pxr import Usd, UsdGeom, UsdPhysics

    import antioch

    stage = antioch.stage()
    bbox_cache = UsdGeom.BBoxCache(Usd.TimeCode.Default(), [UsdGeom.Tokens.default_])
    grid_h, grid_w = grid.shape

    for prim in Usd.PrimRange(stage.GetPrimAtPath("/World")):
        if not prim.HasAPI(UsdPhysics.CollisionAPI):
            continue
        enabled = prim.GetAttribute("physics:collisionEnabled")
        if enabled and enabled.Get() is False:
            continue
        rng = bbox_cache.ComputeWorldBound(prim).ComputeAlignedRange()
        if rng.IsEmpty():
            continue
        mn, mx = rng.GetMin(), rng.GetMax()
        x0 = max(0, int((mn[0] - x_range[0]) / resolution))
        x1 = min(grid_w, int((mx[0] - x_range[0]) / resolution) + 1)
        y0 = max(0, int((mn[1] - y_range[0]) / resolution))
        y1 = min(grid_h, int((mx[1] - y_range[0]) / resolution) + 1)
        grid[y0:y1, x0:x1] = 255
```

## Path 2: `isaacsim.asset.gen.omap` Generator (extension-gated)

The harvested stubs expose `_omap.Generator`, but the module is not importable
in the pinned image until the `isaacsim.asset.gen.omap` extension is enabled.
Enable it before importing the binding, and give Kit a tick to load it:

```python
def generate_omap() -> None:
    import omni.physx
    import omni.usd
    import isaacsim.core.experimental.utils.app as app_utils

    if not app_utils.enable_extension("isaacsim.asset.gen.omap"):
        raise RuntimeError("isaacsim.asset.gen.omap could not be enabled")

    from isaacsim.asset.gen.omap.bindings import _omap

    app_utils.update_app()
    physx = omni.physx.get_physx_interface()
    stage_id = omni.usd.get_context().get_stage_id()
    generator = _omap.Generator(physx, stage_id)
    generator.update_settings(0.1, 4, 5, 6)
    generator.set_transform((0, 0, 0), (-10, -10, 0), (10, 10, 0))
    generator.generate2d()
    buffer = generator.get_buffer()
```

PhysX raycasts see only colliders, so every obstacle needs collision enabled,
the timeline must be playing (`play(commit=True)` before `generate2d()`), and
the origin should be a free point. The stubs describe `set_transform`'s
arguments as an origin plus min/max bounds, and examples pass world-space
values, but the absolute-vs-offset interpretation was **not verified live**
because the binding was unavailable before extension enablement. Treat that
semantics as unverified-on-live and validate it with a known stage. Height
filtering is not a Generator knob; use the direct-projection filters below.

## Path 1 filters: visual bboxes and height bands

### Visual-bbox fallback (scenes without colliders)

Naive bbox projection fills the grid with shell geometry, zones, and
signage. Iterate `UsdGeom.Gprim` instead and filter aggressively:

```python
SKIP_SCOPES = {"GroundPlane", "Looks", "Lighting", "Render"}
SKIP_PREFIXES = ("Floor", "FL_", "FR_", "Exit_", "Hum_")
```

Tailor the lists to the scene — inspect prim names by dumping them once
(`antioch run` a one-off script that prints `stage.TraverseAll()` names).

### Geometric filters (apply after either strategy)

- Skip footprint area > 3000 m² (building shell, zone assemblies).
- Skip height < 0.1 m (floor markings, safety tape — route overlays sit at
  z = 0.02–0.04).
- Skip `z_min > ~3× robot height` (ceiling-only objects, overhead conveyors,
  HVAC).
- Skip `z_max < z_offset + 0.05 m` (anything the robot can drive over —
  `z_offset` from `compute_robot_footprint`, see
  `navigation-footprints-planning.md`).

### Height band

The band between "floor marking" and "above the robot" is what becomes an
obstacle. Typical: keep 0.05–2.0 m; raise the top for tall robots.

## Resolution and buffer sizing

| Use case | Resolution | Grid for a 220×180 m site |
|---|---|---|
| Coarse planning | 0.5 m/px | 440×360 |
| Standard nav | 0.1 m/px | 2200×1800 |
| Fine perception | 0.05 m/px | 4400×3600 |

The dilation buffer is **derived from the footprint**, not a constant:
`circumscribed_radius + safety_margin` (margin table in
`navigation-footprints-planning.md`). Apply with `scipy.ndimage.binary_dilation` and a
circular kernel:

```python
def dilate(grid: "np.ndarray", buffer_m: float, resolution: float) -> "np.ndarray":
    import numpy as np
    from scipy.ndimage import binary_dilation

    buffer_px = int(round(buffer_m / resolution))
    size = 2 * buffer_px + 1
    yy, xx = np.mgrid[:size, :size]
    kernel = (yy - buffer_px) ** 2 + (xx - buffer_px) ** 2 <= buffer_px**2
    return binary_dilation(grid > 0, structure=kernel)
```

## ROS export — map.yaml + PNG

```python
def export_ros_map(grid: "np.ndarray", resolution: float, origin: tuple, out_dir: str) -> None:
    import numpy as np
    import yaml
    from PIL import Image

    img = np.full_like(grid, 205)  # grey = unknown
    img[grid == 0] = 254  # white = free
    img[grid > 0] = 0  # black = occupied
    Image.fromarray(img, mode="L").save(f"{out_dir}/map.png")

    data = {
        "image": "map.png",
        "resolution": resolution,
        "origin": [origin[0], origin[1], 0.0],  # [x, y, yaw] of bottom-left pixel
        "negate": 0,
        "occupied_thresh": 0.65,
        "free_thresh": 0.196,
    }
    with open(f"{out_dir}/map.yaml", "w") as f:
        yaml.dump(data, f, default_flow_style=False)
```

`yaml` and `PIL` are plain Python (no simulator dependency), but keep them
function-scoped anyway — scenario modules must import cleanly with nothing
installed beyond the SDK.

A colored preview (white free / black occupied / pink buffer) is worth
emitting as a run artifact so the exact map remains inspectable after the run:

```python
def export_preview(grid: "np.ndarray", buffered: "np.ndarray", out_path: str) -> None:
    import numpy as np
    from PIL import Image

    color = np.zeros((*grid.shape, 3), dtype=np.uint8)
    color[grid == 0] = [255, 255, 255]
    color[grid > 0] = [0, 0, 0]
    color[buffered & (grid == 0)] = [255, 200, 200]
    Image.fromarray(color, mode="RGB").save(out_path)
```

## Consuming the map

```python
def load_map(path: str) -> None:
    from isaacsim.replicator.experimental.mobility_gen import OccupancyMap

    omap = OccupancyMap.from_ros_yaml(path)
    omap_buffered = omap.buffered_meters(0.5)  # A* planner input
```

## Coordinate conventions

- **USD world**: X = east, Y = north, Z = up (meters).
- **Image**: row 0 = top = max Y (north); col 0 = left = min X (west). Flip
  Y when rasterizing into image rows: `row = grid_h - 1 - world_row`.
- **ROS origin**: `[x, y, yaw]` of the bottom-left pixel in world coords.
- `world_to_pixel`: `col = (world_x - origin_x) / resolution`,
  `row = (max_y - world_y) / resolution`.
- Mind `UsdGeom.GetStageMetersPerUnit(stage)` — a cm-unit stage needs every
  world coordinate scaled before rasterizing.
