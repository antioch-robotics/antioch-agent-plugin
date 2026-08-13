# Import configuration detail — URDF/MJCF → USD

Depth reference for the assets domain of the `isaac-sim-6` skill. Everything here runs
remotely on an Antioch machine after boot; all `pxr` / `isaacsim` imports
stay inside function bodies (the lazy-import invariant).

## Importer config fields

`URDFImporterConfig` / `MJCFImporterConfig` field names, defaults, and
allowed values: `research_search` them (`kind="source"` localizes the config
dataclass). Two fields whose semantics are easy to misread:

- `fix_base` is tri-state: `None` keeps the source's base authoring, `False`
  strips the world→root fixed joint (floating base), `True` adds one and
  relocates `ArticulationRootAPI`.
- `joint_drive_type` / `joint_target_type` / `override_joint_stiffness` /
  `override_joint_damping` each accept a single value or a `{regex: value}`
  per-joint dict.

## Resolving `package://` URLs

The importer maps ROS package names to filesystem paths via
`ros_package_paths=[{"name": "my_pkg", "path": "/abs/path/to/my_pkg"}]`.
The resolved meshes must exist on the machine at those paths. Keep the
description package, including its meshes, inside the project. A declared
`sync` watch rule copies it during interactive work; Antioch adds the submitted
project files to the simulation image for queued runs. Pass the corresponding
remote absolute path to the importer. Publish a robot description that several
projects need as an Antioch asset; `antioch-platform` covers the asset commands.

## `MJCFImporterConfig` notes

Mirrors the URDF config for the shared fields (`merge_mesh`,
`robot_type`, `import_scene`, …). The MJCF-specific surface is actuator
semantics — MuJoCo's general gain/bias model does not map 1:1 to a PhysX
drive, so override it for the control mode you want:

```python
def position_control_overrides(kp: float, kd: float) -> dict:
    return {
        "override_gain_type": "fixed",  # MuJoCo actuator gain type
        "override_bias_type": "affine",  # MuJoCo actuator bias type
        "override_gain_prm": [kp, 0, 0, 0, 0, 0, 0, 0, 0, 0],
        "override_bias_prm": [0, -kp, -kd, 0, 0, 0, 0, 0, 0, 0],
    }
```

Symptom of skipping this: joints hold position in MuJoCo but drift, sag, or
oscillate in PhysX after conversion.

## The asset transformer

With `run_asset_transformer=True` (default), `isaacsim.asset.transformer`
executes an ordered rule pipeline over the import output — physics
conversion (`UrdfToMjcPhysxConversionRule` / `MjcToPhysxConversionRule`)
and material routing. To re-run it on an existing USD:

```python
def run_transformer(input_usd: str, profile_json: str, out_dir: str) -> None:
    from isaacsim.asset.transformer import AssetTransformerManager, RuleProfile

    profile = RuleProfile.from_json(profile_json)  # takes the JSON payload string, not a path
    AssetTransformerManager().run(input_usd, profile, package_root=out_dir)
```

The transformer does not re-run on an existing output: clear the previous
output under `usd_path` (the output directory) before re-importing with a
changed config.

## Robot Schema manual application

The importers apply the schema automatically. For hand-authored USDs, apply
it per prim — or use `PopulateRobotSchemaFromArticulation` for the ordered
relations and add the APIs yourself:

```python
def apply_robot_schema(stage: "Usd.Stage", robot_path: str) -> None:
    from usd.schema.isaac.robot_schema import ApplyJointAPI, ApplyLinkAPI, ApplyRobotAPI, ApplySiteAPI, Attributes
    from usd.schema.isaac.robot_schema.utils import PopulateRobotSchemaFromArticulation

    robot_prim = stage.GetPrimAtPath(robot_path)
    ApplyRobotAPI(robot_prim)
    robot_prim.GetAttribute(Attributes.ROBOT_TYPE.name).Set("Mobile Manipulators")
    PopulateRobotSchemaFromArticulation(stage, robot_prim)
    # ApplyLinkAPI per rigid link, ApplyJointAPI per joint, ApplySiteAPI per
    # grasp/mount frame — Populate fills the ordered relations, not the APIs.
```

Valid `robot_type` tokens (read them back with
`robot_prim.GetAttribute(Attributes.ROBOT_TYPE.name).GetMetadata("allowedTokens")`):
`Default`, `End Effector`, `Manipulator`, `Humanoid`, `Wheeled`, `Holonomic`,
`Quadruped`, `Mobile Manipulators`, `Aerial`.

`IsaacSiteAPI` replaces the deprecated `IsaacReferencePointAPI` — re-import
older assets rather than carrying the deprecated API forward.

Reading back what an imported USD carries:

```python
def inspect_robot_schema(usd_path: str) -> None:
    from pxr import Usd
    from usd.schema.isaac.robot_schema import Attributes, Classes

    stage = Usd.Stage.Open(usd_path)
    robot = next(p for p in stage.Traverse() if p.HasAPI(Classes.ROBOT_API.value))
    print(robot.GetPath(), robot.GetAttribute(Attributes.ROBOT_TYPE.name).Get())
```
