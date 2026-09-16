# Warehouse Occupancy Map Generation

Adapted from NVIDIA [`occupancy-map/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/occupancy-map/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Export ROS-compatible occupancy grids from USD scenes via the omap extension or a USD-projection fallback for Nav2, MobilityGen, and A* planners.

A USD bounding-box projection is only an approximation and can miss descendants or overfill hollow geometry. Grid occupancy is not physical contact. Validate a projection with an asymmetric fixture before treating it as a collision-free map.

Enable `isaacsim.asset.gen.omap` through `SimulationConfig.extensions` before Path 1; installed is not enabled. MobilityGen's Python `OccupancyMap` lives under `isaacsim.replicator.experimental.mobility_gen` (enable that extension ID separately).

## Limitations

- Path 1 requires authored PhysX colliders and a playing timeline; prototype scenes without `CollisionAPI` fall through to USD projection.
- Output is a 2D top-down grid at a single height band; multi-floor or overhanging geometry is not represented.
- Path 2 (projection) ignores physics collision approximations and is only a prototype substitute.

## Troubleshooting

| Error / symptom | Cause | Solution |
|---|---|---|
| `_omap` import fails | Extension not enabled | Add `isaacsim.asset.gen.omap` to `SimulationConfig.extensions`, or use Path 2 |
| Zero occupied cells | Prims lack colliders or timeline not playing | Enable collisions and play through the stepping owner, or use Path 2 |
| Map extent clipped / empty | Wrong origin or lower/upper bounds | Set bounds to cover the facility; origin must be a free cell |

## Upstream helpers (source examples)

| Script | Purpose |
|---|---|
| [`scripts/generate_occupancy_map.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/occupancy-map/scripts/generate_occupancy_map.py) | Unified entry — tries colliders, falls back to USD projection |
| [`scripts/usd_projection_pipeline.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/occupancy-map/scripts/usd_projection_pipeline.py) | Direct USD-projection pipeline (Path 2 only) |

These files are not bundled commands and are not present at a local `scripts/` path in this skill.

## When to use

- Nav2 / MobilityGen / A* path planning setup.
- Perception training data generation.
- AMR fleet path planning validation.
- Collision-avoidance buffer-zone calculation.

## Path 1 (recommended): `isaacsim.asset.gen.omap` extension

Documented in [Mapping](https://docs.isaacsim.omniverse.nvidia.com/latest/digital_twin/ext_isaacsim_asset_generator_occupancy_map.html). The extension uses physics collision geometry, so every prim you want captured must have **Collisions Enabled**; the **Start** location cannot be occupied.

### GUI workflow

1. Open the stage.
2. **Tools > Robotics > Occupancy Map**.
3. Set **Origin** (free point inside the area), **Lower/Upper Bound** (clamp the mapped extent), **Cell Size**, optionally toggle **Use PhysX Collision Geometry**.
4. **CALCULATE**, then **VISUALIZE IMAGE** to preview.
5. From the visualization window: **Save Image** (PNG) and **Save YAML** (ROS occupancy-map parameters file).

GUI Lower/Upper Bound includes the vertical band. That is not the Python `Generator.update_settings` signature.

### Python (programmatic, simulation playing)

Pinned Generator API (`isaacsim.asset.gen.omap.bindings._omap.Generator`): `update_settings(cell_size, occupied_value, unoccupied_value, unknown_value)` then `set_transform(origin, min_bound, max_bound)`. Occupancy values and buffer layout come from those settings, not from keyword `z_min` / `occupancy_threshold` arguments.

```python
from isaacsim.asset.gen.omap.bindings import _omap
import omni.physx
import omni.usd

physx = omni.physx.get_physx_interface()
stage_id = omni.usd.get_context().get_stage_id()

generator = _omap.Generator(physx, stage_id)
# Example values from the pinned stub: cell 0.05 m; occupied=4, unoccupied=5, unknown=6.
# Stage units are assumed meters. Choose values to match the downstream ROS reader.
generator.update_settings(0.05, 4, 5, 6)
generator.set_transform((0, 0, 0), (-10, -10, 0), (10, 10, 0))
generator.generate2d()
buffer = generator.get_buffer()
dims = generator.get_dimensions()
occupied = generator.get_occupied_positions()
```

Requires the timeline to be **playing** for PhysX queries. Pair with the framework stepping owner (`World.step`, `app_utils.play(commit=True)`, or the Lab loop). Origin must be in unoccupied space.

Retrieve buffer orientation from `get_dimensions()` / occupied positions rather than assuming row 0. If internal row zero is minimum world Y and the ROS image's top row is maximum world Y, flip rows once on write.

## Path 2 (fallback): direct USD projection

Use when the scene lacks colliders, you need a deterministic projection from authored geometry, or you are prototyping with placeholder cubes. Faster and reproducible for those cases but does not respect physics collision approximations.

Implementation: [`scripts/usd_projection_pipeline.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/occupancy-map/scripts/usd_projection_pipeline.py).

### 1. Extract obstacles from USD

Read prims, project XY footprint onto a 2D grid. Filter by height to separate navigable floor markings from solid obstacles.

`extract_obstacles_from_usd(...)` in the upstream script returns a uint8 grid. MobilityGen's in-memory convention is `OccupancyMapDataValue`: unknown=0, freespace=1, occupied=2.

### 2. Apply robot buffer (example)

Convert nonnegative clearance to cells with `ceil(clearance_m / resolution_m)`. The radius below is a tuning starting point, not a universal robot radius.

```python
from scipy.ndimage import binary_dilation
import numpy as np

ROBOT_RADIUS = 0.5  # example meters; measure the actual footprint
buffer_px = int(np.ceil(ROBOT_RADIUS / RESOLUTION))
kernel_size = 2 * buffer_px + 1
kernel = np.zeros((kernel_size, kernel_size), dtype=bool)
for r in range(kernel_size):
    for c in range(kernel_size):
        if (r - buffer_px) ** 2 + (c - buffer_px) ** 2 <= buffer_px**2:
            kernel[r, c] = True
buffered = binary_dilation((grid == 2), structure=kernel)
```

Reject out-of-map footprints instead of accepting silent rasterizer clipping. Treat unknown cells according to the declared policy, normally blocked for a collision-free claim.

### 3. Export ROS format

`export_ros_map` in the upstream script writes `map.png` and `map.yaml`. Preserve free, occupied, and unknown values with matching `negate`, `occupied_thresh`, `free_thresh`, resolution, origin, and origin yaw. Load the image/YAML through the downstream reader and check asymmetric world-to-cell samples.

### 4. Generate colored visualization (not ROS evidence)

```python
color = np.zeros((grid_h, grid_w, 3), dtype=np.uint8)
color[grid == 1] = [255, 255, 255]  # white=free
color[grid == 2] = [0, 0, 0]  # black=occupied
color[buffered & (grid != 2)] = [255, 200, 200]  # pink=buffer
Image.fromarray(color, "RGB").save("map_colored.png")
```

## Key design decisions

### What to mark as obstacles (warehouse example)

- **INCLUDE**: racks, docks, tables, pack stations, conveyors with supports, walls, columns, other static structure in the robot height band.
- **EXCLUDE**: floor plane, painted lane markings, moving humans/AMRs, and geometry the robot can drive under or over after measuring the actual height band.

### Height filtering (example starting points)

Declare the vertical obstacle band before generating a grid:

- `z_max < 0.05 m` → floor marking, skip (example: route overlays at z=0.02–0.04)
- `z_min > 2.0 m` → above this robot, skip (example: overhead conveyors)
- Everything else in the declared band → obstacle

MobilityGen defaults in the pin include `OCCUPANCY_MAP_DEFAULT_Z_MIN = 0.1`, `OCCUPANCY_MAP_DEFAULT_Z_MAX = 0.62`, `OCCUPANCY_MAP_DEFAULT_CELL_SIZE = 0.05`.

### Resolution selection (example)

| Use Case | Resolution | Grid Size (220×180 m example) |
|----------|-----------|---------------------|
| Coarse planning | 0.5 m/px | 440×360 |
| Standard nav | 0.1 m/px | 2200×1800 |
| Fine perception | 0.05 m/px | 4400×3600 |

### Robot buffer sizing (example starting points)

Measure the collision footprint; do not treat these as universal radii.

| Robot Type | Example radius | Example buffer |
|-----------|--------|--------|
| Small AMR | 0.3 m | 0.4 m |
| Standard AMR | 0.5 m | 0.6 m |
| Forklift | 1.0 m | 1.2 m |
| Human (walkway planning) | 0.3 m | 0.5 m |

## Isaac Sim OccupancyMap class (MobilityGen)

```python
from isaacsim.replicator.experimental.mobility_gen.impl.occupancy_map import OccupancyMap

omap = OccupancyMap.from_ros_yaml("map.yaml")
omap_buffered = omap.buffered_meters(0.5)
```

Enable `isaacsim.replicator.experimental.mobility_gen` in `SimulationConfig.extensions`. The older `isaacsim.replicator.mobility_gen.impl.occupancy_map` path is not present at this pin. See [mobility-gen.md](mobility-gen.md) and [navigation-primitives.md](navigation-primitives.md).

## Coordinate conventions

- **USD world**: X/Y in the ground plane, Z-up is a common convention, not a USD requirement. Read stage up-axis and `metersPerUnit`.
- **Image**: ROS occupancy images typically have row 0 at the top. If the generator's row 0 is minimum world Y, flip once so the image top is maximum world Y.
- **ROS origin**: `[x, y, yaw]` of the bottom-left pixel in world coords.
- `world_to_pixel` (after the flip): `col = (world_x - origin_x) / resolution`, `row = (origin_y + height_m - world_y) / resolution`.

Keep ROS image orientation and metadata in the same world frame.
