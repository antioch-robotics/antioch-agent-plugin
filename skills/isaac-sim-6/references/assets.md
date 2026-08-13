# Assets — Isaac Sim 6.0.1 on Antioch

Detail reference for the `isaac-sim-6` skill.

Everything below runs remotely on an Antioch GPU machine (Isaac Sim 6.0.1 /
Kit 110.1.2). The laptop has no local simulator or stage, so conversions run as
scripts or scenarios on the machine. You can stream the remote GUI while
debugging. The working loop for conversion:

1. Author the conversion as a scenario or one-off script locally.
2. Dispatch it with `antioch scenario run` / `antioch run` (CLI surface: the
   `antioch-platform` skill — do not guess flags).
3. Under `antioch scenario run` the conversion is a scenario run: keep the
   converted USD (plus collected meshes and materials) with
   `run.add_artifact(...)` and pull it back with
   `antioch scenario download SCENARIO_RUN_ID`. Under `antioch run` the
   script's stdout streams straight to the terminal and no run id exists for
   `scenario logs` / `scenario download`.
4. Commit it to the project, or publish it for the team with
   `antioch assets push`.

Lazy-import invariant: every `pxr` / `omni` / `carb` / `isaacsim` import
lives inside a function body — module scope breaks Antioch's local scenario
discovery. All snippets here follow that. Boot/step/readback ordering, the
6.0 namespace map, and the XformCache trap: this skill's `SKILL.md`.

## Pick the path

| Task | Path |
|---|---|
| Robot URDF → USD | `URDFImporter` (below) |
| Robot MJCF → USD | `MJCFImporter` (below) |
| XACRO source | expand locally with `xacro`, then the URDF path |
| USD → URDF (round-trip back to ROS) | `isaacsim.asset.exporter.urdf.UsdToUrdfConverter` |
| Multi-arm / chassis + tool assembly | USD assembly + validation (below) |
| Retrofit the robot schema on an existing USD | `PopulateRobotSchemaFromArticulation` (below) |
| Measure / place environment assets | measurement section (below) |
| Isaac Lab-native conversion | Lab `convert_urdf.py` + `config.yaml` — see the `isaac-lab-3` skill |

## Stock environments, floors, and scene dressing

Stock USDs are acquired on the GPU machine from the Isaac asset root. The
harvested 6.0.1 examples use these paths:

| Scene | Asset-root-relative USD |
|---|---|
| Full warehouse shell | `/Isaac/Environments/Simple_Warehouse/full_warehouse.usd` |
| Warehouse with multiple shelves | `/Isaac/Environments/Simple_Warehouse/warehouse_multiple_shelves.usd` |
| Warehouse shell variant | `/Isaac/Environments/Simple_Warehouse/warehouse.usd` |
| Simple room | `/Isaac/Environments/Simple_Room/simple_room.usd` |
| Grid room | `/Isaac/Environments/Grid/gridroom_black.usd` or `/Isaac/Environments/Grid/default_environment.usd` |

Reference one of those files in place; do not copy only the top-level USD:

```python
def add_warehouse(stage) -> None:
    import isaacsim.core.experimental.utils.stage as stage_utils
    from isaacsim.storage.native import get_assets_root_path

    root = get_assets_root_path()
    stage_utils.add_reference_to_stage(usd_path=root + "/Isaac/Environments/Simple_Warehouse/full_warehouse.usd", path="/World/Warehouse")
```

These stock scenes carry relative references to meshes, materials, and
sub-assets. A single-file export therefore often opens as a gray shell or an
empty stage. Keep the reference rooted at the machine's Isaac asset tree (or
preserve the complete referenced directory tree); never stage a lone stock
USD and expect its relatives to follow it. For a team-owned asset, use the
Antioch catalog path instead: `antioch.load_asset(name, prim_path=...)`
references the versioned asset into the active stage and keeps acquisition
policy out of scene code (see `antioch-platform`'s assets reference).

For a plain safety floor, the pinned experimental API is enough:

```python
def define_ground_plane() -> None:
    from isaacsim.core.experimental.objects import GroundPlane

    GroundPlane("/World/ground_plane", templates=None)
```

Here `define_ground_plane` is a project-local wrapper name, not an Isaac Sim
6.0.1 symbol in the pinned stubs. Use that wrapper only when it is actually
defined, or use `GroundPlane` directly. A floor mesh without a
collision API is visual dressing, not a physical floor.

The experimental stage helper also has a `create_new_stage(template="gridroom")`
template for a quick blue grid room. It is useful for a camera or contact
smoke test, not a substitute for a textured warehouse when composition is part
of the behavior under test.

## XACRO inputs

The URDF importer core does not parse XACRO. Expand on the user's machine —
`xacro` is pure Python, needs no simulator and no live ROS graph:

```bash
xacro robot.xacro arm_id:=fr3 hand:=true > robot.urdf
```

Commit the expanded URDF so the conversion is reproducible. `package://`
mesh URLs are resolved remotely. Keep the mesh tree inside a declared sync
watch path for interactive work and copy it into the sim image for queued
runs. Pass `ros_package_paths` so the importer can map package names; field
detail lives in `references/assets-import.md`.

## URDF → USD

```python
def convert_urdf(urdf_path: str, usd_path: str) -> str:
    from isaacsim.asset.importer.urdf import URDFImporter, URDFImporterConfig

    config = URDFImporterConfig(
        urdf_path=urdf_path,
        usd_path=usd_path,
        merge_fixed_joints=True,
        fix_base=True,
        collision_from_visuals=True,
        collision_type="Convex Decomposition",
        joint_drive_type="force",
        joint_target_type="position",
        override_joint_stiffness=800.0,
        override_joint_damping=40.0,
        robot_type="Manipulator",
    )
    return URDFImporter(config).import_urdf()
```

Runs after boot inside a scenario or an `antioch run` script. Inside a
scenario, keep the output with `run.add_artifact(usd_file)` on the returned
path so `antioch scenario download SCENARIO_RUN_ID` can pull it back — under
`antioch run` the script's stdout streams to your terminal instead. With `run_asset_transformer=True` (the default) the post-import
`isaacsim.asset.transformer` pipeline restructures the output (physics
conversion, material routing). Note `usd_path` is the output directory; the
return value is the generated USD file. Clear the previous output before
re-importing with a different config, or you may read the old artifact.
Field semantics beyond the defaults: `references/assets-import.md`
(exhaustive field list: `research_search` `URDFImporterConfig`).

`fix_base` by robot type:

| Robot type | `fix_base` |
|---|---|
| Manipulator arm (table/wall mounted) | `True` |
| Mobile (wheeled/holonomic) | `False` |
| Humanoid / legged | `False` |
| Aerial | `False` |

## MJCF → USD

```python
def convert_mjcf(mjcf_path: str, usd_path: str) -> str:
    from isaacsim.asset.importer.mjcf import MJCFImporter, MJCFImporterConfig

    config = MJCFImporterConfig(
        mjcf_path=mjcf_path,
        usd_path=usd_path,
        import_scene=True,
        merge_mesh=True,
        robot_type="Quadruped",
        override_gain_type="fixed",
        override_bias_type="affine",
    )
    return MJCFImporter(config).import_mjcf()
```

MJCF actuators keep MuJoCo gain/bias semantics unless overridden — set
`override_gain_prm` / `override_bias_prm` for the control mode you actually
want, or joints behave wrong under PhysX. The legacy
`MJCFCreateAsset` / `MJCFCreateImportConfig` commands still exist but are
deprecated; use `MJCFImporter` directly. Gain/bias detail: `references/assets-import.md`.

## Drive presets

| Use case | drive type | target | stiffness | damping |
|---|---|---|---|---|
| RL training (policy owns torques) | `None` (no drive) | `none` | 0 | 0 |
| Position teleop / scripted motion | `force` | `position` | 800 | 40 |
| Fine assembly / insertion | `force` | `position` | 100 | 1 |

`joint_drive_type` takes only `"force"`/`"acceleration"` — `"none"` is valid
for `joint_target_type` alone; pass Python `None` to leave drives unauthored.
Every drive field (`joint_drive_type`, `joint_target_type`,
`override_joint_stiffness`, `override_joint_damping`) also accepts a
`{regex: value}` dict for per-joint overrides.

For parallel-env RL, make the converted asset **instanceable** — without
it, N envs × full mesh blows VRAM; with it, all envs share one copy. Isaac
Lab's convert scripts do this via `make_instanceable: true` in
`config.yaml`; with the direct importer, set it where you reference the
converted USD:

```python
prim.SetInstanceable(True)  # on the prim referencing the converted asset
```

## Articulation assembly (multi-arm, chassis + tool)

Two rules, both enforced by the validation script in
`references/assets-validation.md`:

1. Exactly one `UsdPhysics.ArticulationRootAPI` on the whole robot, on the
   chassis — remove it from any referenced limb USD that carries its own.
2. Every limb connects to the chassis through a `FixedJoint` (rigid
   attachment) or an articulated joint chain.

```python
def attach_arm(stage: "Usd.Stage", chassis_path: str, arm_path: str) -> None:
    from pxr import Sdf, UsdPhysics

    joint = UsdPhysics.FixedJoint.Define(stage, f"{arm_path}/AttachmentJoint")
    joint.CreateBody0Rel().SetTargets([Sdf.Path(chassis_path)])
    joint.CreateBody1Rel().SetTargets([Sdf.Path(f"{arm_path}/BaseMount")])
    arm_prim = stage.GetPrimAtPath(arm_path)
    if arm_prim.HasAPI(UsdPhysics.ArticulationRootAPI):
        arm_prim.RemoveAPI(UsdPhysics.ArticulationRootAPI)
```

What does NOT work: referencing the same arm USD twice via sublayer
composition, adding arms as `over` prims in a parts layer, or assuming
visual presence means physical attachment. Visual plausibility is not
mechanical correctness — if it doesn't articulate, it doesn't exist.

## Isaac Robot Schema overlay

The semantic layer (`usd.schema.isaac.robot_schema`) that the importers
apply automatically; tools that walk "this robot's links / joints / poses"
require it (`IsaacRobotAPI` on the root, `IsaacLinkAPI`/`IsaacJointAPI` per
link/joint, `IsaacSiteAPI` for named frames — replacing the deprecated
`IsaacReferencePointAPI`; `research_search` the schema for attribute detail).
Apply it manually to hand-authored or retrofitted USDs.

One-shot retrofit over an existing articulation — fills the ordered
link/joint relations from physics traversal:

```python
def retrofit_robot_schema(stage: "Usd.Stage", robot_path: str) -> None:
    from usd.schema.isaac.robot_schema.utils import PopulateRobotSchemaFromArticulation

    PopulateRobotSchemaFromArticulation(stage, stage.GetPrimAtPath(robot_path))
```

Manual per-prim application and the valid `robot_type` tokens:
`references/assets-import.md`.

## Measuring and placing environment assets

Most USD assets are not origin-centered, and not all are meter-scale.
Measure before placing:

```python
def measure_asset(usd_path: str) -> dict:
    from pxr import Usd, UsdGeom

    # Open the composed stage, not a raw layer: Sdf.Layer.FindOrOpen skips
    # composition, so references/payloads never resolve and world bboxes /
    # default-prim traversal are meaningless.
    stage = Usd.Stage.Open(usd_path)
    mpu = UsdGeom.GetStageMetersPerUnit(stage)
    prim = stage.GetDefaultPrim() or stage.GetPseudoRoot()
    bbox = UsdGeom.BBoxCache(Usd.TimeCode.Default(), [UsdGeom.Tokens.default_])
    extent = bbox.ComputeWorldBound(prim).ComputeAlignedRange()
    lo, hi = extent.GetMin(), extent.GetMax()
    if lo[0] > 1e30:
        raise ValueError(f"Asset '{usd_path}' did not compose — check its references and payloads")
    return {"size_m": tuple((hi[i] - lo[i]) * mpu for i in range(3)), "center": tuple((lo[i] + hi[i]) / 2 for i in range(3)), "base_z": lo[2], "mpu": mpu}
```

Offset-corrected placement (the target is where the bbox center should
land; grounding puts the bbox bottom on z=0):

```text
translate_x = target_x - bbox_center_x
translate_y = target_y - bbox_center_y
translate_z = -bbox_min_z
```

Placement must apply scale first and translation last. Gf uses row vectors
(`A * B` applies `A` first), so the matrix product is `S * R * T` — composing
`T * R * S` scales the translation vector too (a 3 m offset becomes 3 cm at
mpu=0.01). When the asset's
`metersPerUnit` differs from the stage's, wrap it in a parent xform scaled
by mpu instead of editing mesh data. Placement ops belong on the asset's
root prim only (a matrix xform op is the clean choice) — never add xform ops
to child body or link prims, which physics drives. Batch catalog, the placement-matrix helper, and the placement function: `references/assets-measurement.md`.

Remote-rendering shader rule: MDL-only materials come back black in Antioch's
server-side renderer, whether or not the GUI is streaming to the browser.

| Shader authored | Remote-rendering result |
|---|---|
| `UsdPreviewSurface` | renders |
| MDL + `UsdPreviewSurface` fallback | renders (fallback) |
| MDL only | black |

Audit shader prims before committing an asset — detection patterns and the
batch audit: `references/assets-measurement.md`.

## Gotchas (symptom → cause → fix)

| Symptom | Cause | Fix |
|---|---|---|
| OOM in parallel envs | asset not instanceable | `SetInstanceable(True)` / Lab `make_instanceable` |
| Robot flies apart on play | `merge_fixed_joints=False` + stiff PD | merge, or lower gains |
| Wrong masses | URDF inertia missing/ignored | pass `link_density` fallback; verify per-link mass post-import |
| Slow training steps | `allow_self_collision=True` | keep off for training |
| XACRO fails to import | importer does not parse XACRO | expand locally with `xacro` first |
| `package://` URLs unresolved | no mapping or files on the machine | pass `ros_package_paths`; include meshes through a sync watch rule or the sim image |
| MJCF actuators behave wrong | MuJoCo gain/bias defaults kept | set `override_gain_type` / `override_bias_prm` |
| Transformer output ignored | stale USD at `usd_path` | delete output, re-import with `run_asset_transformer=True` |
| Arms render but float | no `FixedJoint` to chassis | add joint; remove arm's `ArticulationRootAPI` |
| Multiple articulation roots | limb USD carries its own root | `RemoveAPI(UsdPhysics.ArticulationRootAPI)` on limbs |
| Poser/IK tools error out | missing `IsaacRobotAPI` / relations | `PopulateRobotSchemaFromArticulation` |
| `robot_type` value rejected | typo or stale token | `GetAttribute(Attributes.ROBOT_TYPE.name).GetMetadata("allowedTokens")` |
| Asset 100× too small | cm asset (mpu=0.01) treated as meters | read mpu; scale a wrapper xform |
| Asset lands offset from target | bbox not origin-centered | offset-corrected placement |
| Black renders headless | MDL-only material | use dual-shader / PreviewSurface assets |
| bbox min > 1e30 | broken reference or payload | fix composition, then re-measure |

## References (load one level deep when needed)

- `references/assets-import.md` — importer-config semantics worth knowing
  (`fix_base` tri-state, per-joint regex overrides), the asset transformer
  pipeline, manual Robot Schema application, `robot_type` tokens, and
  `ros_package_paths` mapping. Load when tuning an import beyond the
  defaults; exhaustive config fields: `research_search` them.
- `references/assets-validation.md` — the articulation validation checklist and a
  complete runnable validation script. Load before trusting any imported or
  assembled robot.
- `references/assets-measurement.md` — batch asset catalog, the T·R·S placement
  helper, offset correction, and the headless shader audit. Load when
  measuring or placing environment assets at scale.
