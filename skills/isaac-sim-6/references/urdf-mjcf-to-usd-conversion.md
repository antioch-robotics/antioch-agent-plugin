# URDF / MJCF -> USD Conversion

Adapted from NVIDIA [`urdf-mjcf-to-usd-conversion/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/urdf-mjcf-to-usd-conversion/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Import URDF and MJCF robot descriptions to USD with modern importer APIs, instanceable meshes, and drive configuration for RL or teleop.

Enable `isaacsim.asset.importer.urdf`, `isaacsim.asset.importer.mjcf`, and/or `isaacsim.asset.exporter.urdf` through `SimulationConfig.extensions` before import; installed is not enabled. A Python module path is not always the extension ID.

See also [assets.md](usd-pipeline.md), [usd-articulation.md](usd-articulation.md), [usd-composition-architecture.md](usd-composition-architecture.md).

Two conversion paths and one export path.

| Path | When | Driver |
|---|---|---|
| 1. Isaac Sim importer | RL/Lab asset, scripted batch | `isaacsim.asset.importer.urdf` / `.mjcf` (`URDFImporter` / `MJCFImporter`) |
| 2. Isaac Lab convert script | Isaac Lab-native config.yaml workflow | Lab `scripts/tools/convert_urdf.py` / `convert_mjcf.py` (isaac-lab-3; no local `$ISAAC_LAB_DIR`) |
| Export | USD -> URDF round-trip | `isaacsim.asset.exporter.urdf` |

## Upstream helpers (source examples)

| Script | Purpose |
|---|---|
| [`scripts/urdf_importer_ros.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/urdf-mjcf-to-usd-conversion/scripts/urdf_importer_ros.py) | Import URDF from a live ROS 2 `robot_state_publisher` via `URDFImporter` |

Not a bundled command. ROS 2 runtime and workspace sourcing belong to [antioch-platform](../../antioch-platform/SKILL.md); enable `isaacsim.ros2.urdf` / `isaacsim.ros2.bridge` through `SimulationConfig.extensions`.

## XACRO inputs

The `URDFImporter` core does not parse XACRO. Two supported paths:

### Recommended — import directly from a running ROS 2 node

`isaacsim.ros2.urdf` adds a dedicated import path that queries the `robot_description` parameter on any node (typically `robot_state_publisher`) via the standard `GetParameters` service, resolves `package://` URLs, writes the URDF to a temp file, and feeds it to `URDFImporter`. The node is responsible for XACRO expansion, so this also covers launch-file-only distributions that never ship a static URDF.

UI: `File -> Import from ROS2 URDF Node` (opens an import window with the same collider / robot-type / mesh options as the standard URDF importer).

Python (preferred over the deprecated `URDFImportFromROS2Node` Kit command):

`import_urdf_from_ros(usd_out_path, merge_fixed_joints, fix_base, robot_type)` — subscribe to `robot_state_publisher`, resolve `package://` URLs, and import URDF when the description is received.

See [`scripts/urdf_importer_ros.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/urdf-mjcf-to-usd-conversion/scripts/urdf_importer_ros.py).

Requires the `isaacsim.ros2.urdf` extension (depends on `isaacsim.ros2.bridge` for the ROS 2 runtime), a reachable node publishing `robot_description`, and a session where the robot workspace packages are available. The workspace must include the robot description package and every package referenced by `package://` mesh/resource URLs; the bundled ROS 2 runtime does not provide robot-specific packages. The reader runs asynchronously — the callback fires once the `GetParameters` service replies. The authoring client's shell environment is not the remote service environment.

### Fallback — offline xacro CLI

`URDFImporter` does not parse XACRO. Expand to a static `.urdf` first (the `xacro` CLI, or a launch-file node that already expands it). ROS workspace sourcing and `package://` resolution belong to [antioch-platform](../../antioch-platform/SKILL.md); do not `pip install` or `apt install` on this workstation.

```bash
xacro robot.xacro > robot.urdf
xacro robot.xacro arm_id:=fr3 hand:=true > robot.urdf
```

Pass the resulting `.urdf` to Path 1 or Path 2. `ros_package_paths` on `URDFImporterConfig` maps `package://` names to directories when there is no live ROS graph.

## Path 1 — Isaac Sim importer (recommended for RL/Lab)

Modern public API: `isaacsim.asset.importer.urdf.URDFImporter` + `URDFImporterConfig` (dataclass) and the matching `isaacsim.asset.importer.mjcf` pair. The post-import `isaacsim.asset.transformer` runs by default and restructures the USD output (collects dependencies, runs registered rules for physics conversion, materials routing, etc.).

```python
from isaacsim.asset.importer.urdf import URDFImporter, URDFImporterConfig

config = URDFImporterConfig(
    urdf_path="/path/robot.urdf",
    usd_path="/path/out",
    merge_fixed_joints=True,
    fix_base=False,  # False = floating-base; None = leave source authoring
    collision_from_visuals=True,
    collision_type="Convex Decomposition",
    joint_drive_type="force",
    joint_target_type="position",
    override_joint_stiffness=800.0,  # example Nm/rad; retune
    override_joint_damping=40.0,  # example
    robot_type="Manipulator",  # robot-schema token
    run_asset_transformer=True,  # default True; applies transformer profile
    run_multi_physics_conversion=True,  # URDF -> PhysX/MuJoCo physics
)
output_usd = URDFImporter(config).import_urdf()
```

### `URDFImporterConfig` fields (defaults)

| Field | Default | Notes |
|---|---|---|
| `urdf_path`, `usd_path` | `None` | input/output |
| `merge_fixed_joints` | `False` | collapse fixed joints |
| `merge_mesh` | `False` | merge meshes per link |
| `debug_mode` | `False` | extra logging + intermediates |
| `collision_from_visuals` | `False` | derive collision geom from visuals |
| `collision_type` | `"Convex Hull"` | `Convex Hull` / `Convex Decomposition` / `Bounding Sphere` / `Bounding Cube` |
| `allow_self_collision` | `False` | leave off for training |
| `ros_package_paths` | `[]` | resolve `package://` URLs |
| `robot_type` | `"Default"` | robot-schema token; see below |
| `fix_base` | `None` | `True`: add world→root fixed joint and relocate `ArticulationRootAPI`. `False`: floating-base (remove world→root fixed joint). `None`: leave source authoring |
| `link_density` | `None` | kg/m^3 fallback when URDF has no mass |
| `joint_drive_type` | `None` | `force` / `acceleration`; or `{regex: value}` per-joint |
| `joint_target_type` | `None` | `none` / `position` / `velocity`; or per-joint dict |
| `override_joint_stiffness` | `None` | Nm/rad (rev) or N/m (pris); or per-joint dict |
| `override_joint_damping` | `None` | Nm*s/rad / N*s/m; or per-joint dict |
| `run_asset_transformer` | `True` | run transformer profile post-import |
| `run_multi_physics_conversion` | `True` | URDF -> PhysX joint attr conversion |

### CLI example (native Isaac Sim tree)

The upstream standalone example `source/standalone_examples/api/isaacsim.asset.importer.urdf/urdf_import.py` auto-enables `omni.scene.optimizer.core` and `isaacsim.robot.schema`, then applies the config. It is a source example in the Isaac Sim tree, not a local `python.sh` command on Antioch. Prefer the Python `URDFImporter` API after `antioch.start_simulation()` (plain script) or via `SimulationConfig` on a managed scenario.

Equivalent flags on that example: `--urdf`, `--usd-path`, `--merge-fixed-joints`, `--fix-base`, `--joint-drive-type force`, `--joint-target-type position`, `--collision-from-visuals`, `--collision-type "Convex Decomposition"`, `--robot-type Manipulator`, `--ros-package NAME:PATH`, `--no-run-asset-transformer`.

`--robot-type` choices come from `usd.schema.isaac.robot_schema.get_allowed_tokens(Attributes.ROBOT_TYPE)`: `Default`, `End Effector`, `Manipulator`, `Humanoid`, `Wheeled`, `Holonomic`, `Quadruped`, `Mobile Manipulators`, `Aerial`.

### MJCF import (parallel API)

```python
from isaacsim.asset.importer.mjcf import MJCFImporter, MJCFImporterConfig

config = MJCFImporterConfig(
    mjcf_path="/path/robot.xml",
    usd_path="/path/out",
    import_scene=True,  # include MJCF scene settings
    merge_mesh=True,
    robot_type="Quadruped",
    override_gain_type="fixed",  # MuJoCo actuator gain type
    override_bias_type="affine",  # MuJoCo actuator bias type
    override_gain_prm=[kp, 0, 0, 0, 0, 0, 0, 0, 0, 0],  # position control
    override_bias_prm=[0, -kp, -kd, 0, 0, 0, 0, 0, 0, 0],  # position control
)
output_usd = MJCFImporter(config).import_mjcf()
```

Matching MJCF standalone example: `source/standalone_examples/api/isaacsim.asset.importer.mjcf/mjcf_import.py` (`--mjcf`, `--usd-path`, `--import-scene`, `--robot-type`, `--override-gain-type`, `--override-bias-type`, etc.). Source example, not a bundled Antioch command.

Legacy MJCF commands `MJCFCreateAsset` / `MJCFCreateImportConfig` are deprecated; use `MJCFImporter` directly.

> **Migration:** for the broader `omni.importer.mjcf` / `omni.importer.urdf` → `isaacsim.asset.importer.*` rename map, see [Renaming Extensions](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_4_5/extensions_renaming.html).

### Asset Transformer (what `run_asset_transformer=True` runs)

`isaacsim.asset.transformer` executes ordered USD rule pipelines. Default post-import rules include `UrdfToMjcPhysxConversionRule` / `MjcToPhysxConversionRule` and material routing. To run manually on an existing USD:

```python
from isaacsim.asset.transformer import AssetTransformerManager, RuleProfile

manager = AssetTransformerManager()
profile = RuleProfile.from_json("path/to/profile.json")
report = manager.run("input.usd", profile, package_root="/tmp/out")
```

Standalone example (not a bundled command): `source/standalone_examples/api/isaacsim.asset.transformer/run_asset_transformer.py` (`--input`, `--profile`, `--output`).

### Robot Schema applied during import

The importer applies the modern `usd.schema.isaac.robot_schema`:

- `IsaacRobotAPI` on the robot root prim (stores `robot_type`, ordered link/joint relations, named-pose container).
- `IsaacLinkAPI` on rigid links.
- `IsaacJointAPI` on joints.
- `IsaacSiteAPI` on sites (replaces deprecated `IsaacReferencePointAPI`).
- `IsaacNamedPose` prims for named poses; manage via the `isaacsim.robot.poser` module (see [manipulation-ik.md](manipulation-ik.md)).

Validate after import:

```python
from pxr import Usd
from usd.schema.isaac.robot_schema import Classes, get_allowed_tokens, Attributes

stage = Usd.Stage.Open("/path/out/robot.usd")
robot = next(p for p in stage.Traverse() if p.HasAPI(Classes.ROBOT_API))
print(robot.GetAttribute(Attributes.ROBOT_TYPE).Get())
```

## Path 2 — Isaac Lab `convert_urdf.py` / `convert_mjcf.py` (config.yaml)

For Isaac Lab-native workflows you have a `config.yaml` per robot under `assets/isaaclab/Robots/<Robot>/`:

```yaml
asset_path: /path/to/robot.urdf      # pre-expand XACRO first
usd_file_name: robot_name.usd

force_usd_conversion: true
make_instanceable: true              # critical for parallel envs
import_inertia_tensor: true          # use URDF inertia
merge_fixed_joints: true
self_collision: false

fix_base: false
default_drive_type: none             # "none" | "position" | "velocity"
default_drive_stiffness: 0.0
default_drive_damping: 0.0

link_density: 0.0
convex_decompose_mesh: false
```

Run the Lab converters through the isaac-lab-3 skill / Lab Python environment in the remote session, not a local `./isaaclab.sh` or `$ISAAC_LAB_DIR`. Equivalent entry points: `scripts/tools/convert_urdf.py` and `scripts/tools/convert_mjcf.py` with `--config` and `--output`.

### Drive presets

| Use case | Drive config |
|---|---|
| RL training (agent controls torques) | `default_drive_type: none`, stiffness `0.0`, damping `0.0` |
| Position-controlled teleop | `default_drive_type: position`, stiffness `800.0`, damping `40.0` (example gains) |
| Assembly / manipulation objects (AutoMate) | `default_drive_type: force`, `joint_drive.target_type: position`, `gains: {stiffness: 100, damping: 1}` (example), `collider_type: convex_hull` |

### `make_instanceable: true`

GPU mesh instancing. Without it, 4096 envs * full mesh = VRAM blow-up. With it, parallel envs share one mesh in VRAM. Always set for RL.

### `fix_base` by robot type

| Robot type | `fix_base` |
|---|---|
| Manipulator arm (table/wall mounted) | `true` |
| Mobile robot (wheels) | `false` |
| Humanoid / legged | `false` |
| Aerial (drone) | `false` |

## URDF export (USD -> URDF round-trip)

`isaacsim.asset.exporter.urdf` is the inverse path; useful for sharing imported USD assets back to ROS or other URDF-consumers.

```python
from pxr import Usd
from isaacsim.asset.exporter.urdf import UsdToUrdfConverter

stage = Usd.Stage.Open("/path/robot.usd")
converter = UsdToUrdfConverter(
    stage, root_prim_path="/World/robot", mesh_dir_name="meshes", mesh_path_prefix="./", visualize_collision_meshes=False, variant_selections=None
)
converter.export("/path/urdf_out")
```

Standalone example (not a bundled command): `source/standalone_examples/api/isaacsim.asset.exporter.urdf/urdf_export.py` (`--usd-path`, `--output-dir`, `--root-prim`, `--mesh-prefix`, `--variant SET=SELECTION`).

## Common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| OOM during training | `make_instanceable: false` | set `true` |
| Robot flies apart | `merge_fixed_joints: false` + stiff PD | set true or reduce gains |
| Wrong masses | `import_inertia_tensor: false` + bad geometry | set `true` |
| Self-collision slowdown | `self_collision: true` during training | set `false` |
| XACRO not converting | `URDFImporter` core does not parse XACRO | import from a running node via `isaacsim.ros2.urdf` (`RobotDefinitionReader` / `File -> Import from ROS2 URDF Node`), or pre-expand offline with `xacro robot.xacro > robot.urdf` |
| `package://` URLs unresolved | missing mapping | pass `ros_package_paths=[{"name":..., "path":...}]` or `--ros-package NAME:PATH` |
| MJCF actuators behave wrong | gain/bias type left at MJCF default | set `override_gain_type` / `override_bias_type` / `override_gain_prm` / `override_bias_prm` |
| Transformer output ignored | running an old test asset | delete `usd_path` and re-import with `run_asset_transformer=True` |

Importer/exporter extensions are also published as standalone pip wheels for non-Kit consumers (`isaacsim-asset-importer-urdf`, `isaacsim-asset-importer-mjcf`, `isaacsim-asset-exporter-urdf`, `isaacsim-asset-transformer`). Do not build wheels with `./repo.sh` on this workstation; use the remote image's installed extensions.
