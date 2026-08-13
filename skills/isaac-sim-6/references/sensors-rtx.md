# RTX lidar / radar / acoustic (Isaac Sim 6.0.1, on Antioch)

Non-camera RTX sensors in `isaacsim.sensors.experimental.rtx`. All snippets
obey the lazy-import invariant. The GMO Writer consumption pattern these
sensors share is in `references/sensors.md`; this file holds the vendor
catalog, mount attachment, custom scan patterns, GMO field layouts, and
modality specifics.

Vendor assets resolve against the machine's configured Isaac asset root via
`isaacsim.storage.native.get_assets_root_path()` — never hardcode an asset
URL or root into scenario code; `Lidar.create(config=...)` resolves it for
you.

## `SUPPORTED_LIDAR_CONFIGS` — the vendor catalog

`isaacsim.sensors.experimental.rtx.SUPPORTED_LIDAR_CONFIGS` is the official
list of vendor lidar/radar USD assets (NVIDIA examples,
Ouster, HESAI, SICK, Slamtec, ZVISION). Keys are asset paths under
`get_assets_root_path() + "/Isaac/Sensors/..."`; values are either a `set` of
flat variant names (against the `"sensor"` variant set, named by the
companion constant `SUPPORTED_LIDAR_VARIANT_SET_NAME`) or a list of explicit
`{variant_set: value}` dicts. For the model list and exact variant strings:
`research_search` the registry (`kind="source"`), or probe it on a machine
with `antioch run` — do not guess variants from the asset filename.

## `Lidar.create`

```python
def make_lidar() -> None:
    from isaacsim.sensors.experimental.rtx import Lidar

    lidar = Lidar.create(
        path="/World/Robot/sensor_mount/lidar",
        config="OS1",  # stem alias matched against the registry's asset-path keys
        variant="OS1_REV6_32ch20hz512res",  # required for multi-variant assets
        aux_output_level="FULL",
        tick_rate=20.0,
        accumulate_outputs=True,
    )
```

- `config` is a stem alias matched against the `SUPPORTED_LIDAR_CONFIGS`
  asset-path keys (e.g. `OS1` → `/Isaac/Sensors/Ouster/OS1/OS1.usd`).
- `variant` is **required** for assets exposing multiple variants (Ouster
  OS*, most SICK models, …); the registry value tells you the set.
- `accumulate_outputs=True` delivers a full scan per output frame; `False`
  streams partial packets.

## Attaching a vendor sensor to a robot mount

Author a mount xform under the robot, reference the vendor USD beneath it,
select the variant. In USDA (authored scene assets):

```usda
over "Robot" {
    def Xform "sensor_mount" {
        double3 xformOp:translate = (0.0, 0.0, 0.35)
        uniform token[] xformOpOrder = ["xformOp:translate"]

        def "lidar" (
            prepend references = @</path/from/assets_root/Isaac/Sensors/Ouster/OS1/OS1.usd>@
            variants = { string sensor = "OS1_REV6_32ch20hz512res" }
        ) {}
    }
}
```

In Python (the usual Antioch path — scenes are built in scenario code):

```python
def attach_lidar(stage) -> None:
    from isaacsim.storage.native import get_assets_root_path
    from pxr import Gf, UsdGeom

    assets_root = get_assets_root_path()
    mount = UsdGeom.Xform.Define(stage, "/World/Robot/sensor_mount")
    UsdGeom.Xformable(mount.GetPrim()).AddTranslateOp().Set(Gf.Vec3d(0, 0, 0.35))

    sensor_prim = stage.DefinePrim("/World/Robot/sensor_mount/lidar")
    sensor_prim.GetReferences().AddReference(assets_root + "/Isaac/Sensors/Ouster/OS1/OS1.usd")
    vset = sensor_prim.GetVariantSets().GetVariantSet("sensor")
    vset.SetVariantSelection("OS1_REV6_32ch20hz512res")
```

`Lidar.create(path=<mount path>, config=..., variant=...)` does both steps and
returns the wrapper — prefer it unless the mount lives in an authored USDA
asset.

## Custom scan patterns (emitter-state arrays)

For sensors not in `SUPPORTED_LIDAR_CONFIGS`, author a prim with
`OmniSensorGenericLidarCoreAPI` and emitter-state arrays. Each beam is one
azimuth / elevation / fire-time entry; solid-state patterns run into the tens
of thousands of entries.

```usda
def OmniLidar "Lidar" (
    prepend apiSchemas = ["OmniSensorGenericLidarCoreAPI", "OmniSensorGenericLidarCoreEmitterStateAPI:s001"]
)
{
    float omni:sensor:Core:nearRangeM = 0.1
    float omni:sensor:Core:farRangeM = 120.0
    float omni:sensor:Core:minAzimuthROI = 0.0
    float omni:sensor:Core:maxAzimuthROI = 360.0

    # Emitter state arrays define exact beam directions and timing — the
    # azimuth/elevation arrays carry the FOV and angular resolution.
    float[] omni:sensor:Core:emitterState:s001:azimuthDeg = [0.0, 0.1, 0.2, ...]
    float[] omni:sensor:Core:emitterState:s001:elevationDeg = [-22.5, -20.0, ...]
    int[] omni:sensor:Core:emitterState:s001:fireTimeNs = [0, 55, 110, ...]
}
```

Wrap a custom prim from Python with `Lidar("/World/.../my_lidar")` — no
`config=` argument; the constructor wraps the existing `OmniLidar` prim
instead of creating one.

## GMO buffers and decode helpers

Every RTX sensor emits a `GenericModelOutput` (GMO) buffer consumed through a
`Writer` (pattern in `references/sensors.md`). Decode with
`parse_generic_model_output_data` (raw GMO bytes → structured object with
`numElements`, `x`, `y`, `z`, `scalar`, `matId`, …); the other helpers in
`isaacsim.sensors.experimental.rtx` (stable-ID maps, object IDs, debug image
rendering): `research_search` them. GMO field semantics are
modality-specific:

- **Lidar** — `x`/`y`/`z` are hit coordinates; `scalar` carries intensity;
  `matId`/object IDs identify what was hit (needs `aux_output_level` above
  `NONE`).
- **Radar** — hit position plus radial velocity in the aux channels.
- **Acoustic** — encodes signal ways: `x` = transmitter mount ID, `y` =
  receiver mount ID, `z` = channel ID, `scalar` = amplitude.

For remote debugging, `draw-point-cloud` is a built-in writer that visualizes
the scan — useful in a streamed Jupyter session or scenario, unnecessary in a
headless run:
`sensor.attach_writer("draw-point-cloud", size=0.05, color=[0, 1, 0.5, 1.0])`.

## Radar

```python
def make_radar() -> None:
    import carb.settings

    from isaacsim.sensors.experimental.rtx import Radar, RadarSensor

    # Motion BVH must be on BEFORE Radar(...) — with
    # /renderer/raytracingMotion/enabled off, creation raises RuntimeError.
    carb.settings.get_settings().set_bool("/renderer/raytracingMotion/enabled", True)

    radar = Radar("/World/radar", tick_rate=20.0, aux_output_level="BASIC")
    sensor = RadarSensor(radar, annotators=[])
```

Attach a GMO writer exactly as for lidar.

## Acoustic (ultrasonic)

```python
def make_acoustic() -> None:
    from isaacsim.sensors.experimental.rtx import Acoustic, AcousticSensor

    acoustic = Acoustic(
        "/World/acoustic",
        tick_rate=30.0,
        aux_output_level="BASIC",
        attributes={"omni:sensor:WpmAcoustic:centerFrequency": 51200.0, "omni:sensor:WpmAcoustic:sensorMount:m001:position": (0.0, 0.0, 0.0)},
    )
    sensor = AcousticSensor(acoustic, annotators=[])
```

Same GMO writer consumption as lidar; field layout above.

## Gotchas

- `Lidar.create` without `variant=` on a multi-variant asset raises — read the
  variant set from `SUPPORTED_LIDAR_CONFIGS` first.
- `aux_output_level` is modality-specific: lidar `NONE`/`BASIC`/`EXTRA`/`FULL`;
  radar and acoustic only `NONE`/`BASIC`. Cameras produce no GMO, so
  `RtxCamera` has no `aux_output_level` parameter at all.
- RTX sensors auto-enable their backing `omni.sensors.nv.lidar` / `.nv.radar`
  / `.nv.acoustic` extensions when `isaacsim.sensors.experimental.rtx` loads —
  no manual extension management.
- GMO consumers must be writers; per-step `get_data("generic-model-output")`
  polling under multitick drops or duplicates frames (see
  `references/sensors.md`).
