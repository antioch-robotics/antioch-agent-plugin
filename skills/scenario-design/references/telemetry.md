# Rerun telemetry and blueprints

Context: read this when authoring a viewer layout, wiring live telemetry, or
diagnosing a recording that does not look right. `SKILL.md` owns the scenario
model and the telemetry defaults; this file owns the Rerun surface underneath.

The SDK and browser viewer are pinned together at `rerun-sdk==0.36.0` and
`@rerun-io/web-viewer==0.36.0`. Every constructor below is verified in that
release.

## Live versus recorded

Every `ScenarioSession` opens a file sink. An Antioch scenario finalizes that
file and uploads it under the reserved `telemetry` artifact (content type
`application/vnd.rerun.rrd`). That durable `.rrd` is the run's evidence and it
exists whether or not anyone watched.

Live telemetry is a second sink served by the Rerun gRPC server on the machine,
only while the session is live. The same lease also starts the Isaac WebRTC
viewport: it is the machine's one scarce livestream, not a property of the
project. A "stream available" badge means a machine can accept the
lease, not that anything is streaming.

| Need | Use | Important detail |
| --- | --- | --- |
| Inspect a running authored scenario | `antioch scenario run --stream` | The runner creates the live sink and `run.live_uri`; it closes at scenario end. |
| Run a one-off native script with the GUI | `antioch run --stream FILE` | The CLI reserves the lease. A script that also wants Rerun telemetry must open `ScenarioSession(..., live=True)`. |
| Iterate in a Jupyter kernel | `antioch jupyter stream --kernel KERNEL_ID --json`, then boot | Declare the lease before the first `antioch.boot()`; keep one live `ScenarioSession` across telemetry cells. |
| Review or share a finished result | `antioch scenario download SCENARIO_RUN_ID` | Open the downloaded `.rrd`; no live lease needed. |

```text
antioch.ScenarioSession(name: str, *, rrd_path: str | Path, control: ScenarioControl | None = None,
                        live: bool = False, blueprint: rrb.Blueprint | None = None,
                        capture: bool = True, case_id: str | None = None,
                        tags: tuple[str, ...] = (),
                        params: dict[str, object] | None = None) -> None
ScenarioRun.live_uri -> str | None
```

`live` controls the optional Rerun sink, `capture` disables the platform's
viewport read-back for that session, and `blueprint` seeds the layout —
`run.set_blueprint` is the canonical way to select one. `live_uri` is `None`
when nothing is serving.

There is no `rerun.connect` in this flow, and no such attribute in the pinned
SDK. Antioch creates the `RecordingStream`, the file sink, and the optional
live sink. User code logs, and selects a blueprint.

## Entity paths

`Logger` paths are *relative*: alphanumeric segments plus `_`, `.`, or `-`,
separated by `/`, no leading or trailing slash, no empty segment. The logger's
prefix is prepended.

```python
logger = antioch.Logger("robot")
logger.scalar("metrics/speed_mps", speed)  # entity: /robot/metrics/speed_mps
logger.image("camera/front", rgb)  # entity: /robot/camera/front
logger.value("metrics", {"error": e, "reward": r})  # child series per key
```

Blueprint `origin` and `contents` use Rerun's *absolute* spelling (`/robot`,
`/robot/**`). Keep a stable hierarchy — `robot/`, `camera/`, `metrics/`,
`scene/` — so a layout selects it without knowing per-run values.

`Logger.value` writes a Rerun archetype through unchanged, turns a non-empty
mapping of numbers into child scalar series, and falls back to a text log for
anything else. Use `Logger.image` for pictures and `Logger.scalar` for numbers;
`value` is for the rest — transforms, boxes, point clouds, meshes.

`Logger` is explicit and time-local. A call writes one sample at the current
`sim_time`. It does not inspect the USD stage, turn prims into Rerun geometry,
or backfill the time before that call. Rerun keeps the latest sample visible
after it exists, but an entity first logged at 3 s is absent before 3 s.

Make the first useful rendered step a complete review state: log the owned
camera, drawable scene geometry, and baseline metrics together after reset and
camera setup. Then log on meaningful task changes. Do not copy a fixed warm-up
count or cadence from another scenario; the task decides when its evidence
changes.

A transform is not geometry. `Transform3D` only positions child entities; it
does not draw a robot or prop. Log `Boxes3D`, `Points3D`, `Mesh3D`, or another
drawable archetype at the scene path when a 3D pane must show an object.

```python
logger.value("base", rr.Transform3D(translation=[0.0, 0.0, 0.5], quaternion=[0.0, 0.0, 0.0, 1.0], parent_frame="World", child_frame="robot/base"))
logger.value("scene/obstacle", rr.Boxes3D(centers=[[1.0, 0.0, 0.5]], sizes=[[0.5, 0.5, 1.0]], colors=[[255, 128, 0]]))
logger.value("points", rr.Points3D([[0.0, 0.0, 0.5], [1.0, 0.0, 0.5]], radii=[0.04, 0.04]))
```

## Blueprints

```python
import rerun.blueprint as rrb
```

```text
ScenarioRun.set_blueprint(blueprint: rrb.Blueprint) -> None
```

Call it while the run is active — before or after logging; it raises
`StateError` afterwards. `@antioch.scenario(blueprint=...)` and
`ScenarioSession(blueprint=...)` take the same value for a constant layout, but
`set_blueprint` works everywhere and can react to what the run discovered.

Two rules the viewer enforces silently:

1. **Name a container as the root.** `rrb.Blueprint(*views)` serializes a root
   with `container_kind = Tabs`, so the recording opens showing one pane and
   hiding the rest. Use `rrb.Grid`, `rrb.Horizontal`, or `rrb.Vertical`.
2. **An author blueprint replaces the automatic one entirely.** Every entity
   you want visible needs a view, `/antioch/viewport` included. Naming any view at all
   turns the viewer's own automatic layout off.

The SDK supplies a hidden time panel on `sim_time` when the author omits one.
Add a panel only when the reviewer needs it, and give it an explicit state.
Select `wall_time` only when the evidence explicitly concerns process latency
rather than simulated motion.

`SpatialInformation` is a Rerun blueprint archetype, not an Antioch class, and
it has no default frame:

```text
rrb.SpatialInformation(
    target_frame: str,
    *,
    show_axes: bool | None = None,
    show_bounding_box: bool | None = None,
) -> SpatialInformation
```

`target_frame` resolves the transforms a 3D view shows. Match it exactly to the
frame IDs in your `Transform3D(parent_frame=..., child_frame=...)` calls. If
your transform graph uses Rerun's TF namespace the ID may be `"tf#/"` rather
than `"World"`.

### A complete dashboard

```python
import antioch
import rerun.blueprint as rrb

ROBOT_DASHBOARD = rrb.Blueprint(
    rrb.Horizontal(
        rrb.Vertical(
            rrb.Spatial2DView(origin="/robot/camera/front", name="Bench camera"),
            rrb.Spatial3DView(
                origin="/robot", contents="/robot/**", name="Robot", spatial_information=rrb.SpatialInformation(target_frame="World", show_axes=True)
            ),
        ),
        rrb.TimeSeriesView(origin="/robot/metrics", contents="/robot/metrics/**", name="Metrics"),
    )
)


@antioch.scenario(capture=False)
def robot_dashboard(run: antioch.ScenarioRun) -> None:
    import rerun as rr
    import numpy as np
    from isaacsim.core.api.objects import FixedCuboid
    from isaacsim.core.utils.prims import create_prim
    from isaacsim.core.utils.viewports import set_camera_view

    run.set_blueprint(ROBOT_DASHBOARD)
    logger = antioch.Logger("robot")
    world = antioch.world()
    create_prim("/World/dome_light", "DomeLight", attributes={"inputs:intensity": 400.0})
    create_prim("/World/key_light", "DistantLight", attributes={"inputs:intensity": 1500.0})
    world.scene.add(
        FixedCuboid(
            prim_path="/World/ReviewBody",
            name="review_body",
            position=np.array([0.0, 0.0, 0.25]),
            size=1.0,
            scale=np.array([0.4, 0.3, 0.5]),
            color=np.array([0.9, 0.2, 0.1]),
        )
    )
    world.reset()
    set_camera_view(eye=[1.2, 1.2, 0.9], target=[0.0, 0.0, 0.25], camera_prim_path="/OmniverseKit_Persp")
    world.step(render=True)
    frame = antioch.capture_viewport()
    assert frame is not None, "active viewport returned no frame"
    rgb = np.asarray(frame)[..., :3]
    mean, std = float(rgb.mean()), float(rgb.std())
    subject_pixels = int(((rgb[..., 0] > 1.2 * rgb[..., 2]) & (rgb[..., 0] > 50)).sum())
    exposure_ok = 10 <= mean <= 220
    contrast_ok = std >= 5
    subject_ok = subject_pixels >= 50
    run.check("review frame has useful exposure", exposure_ok, detail=f"mean rgb {mean:.1f}")
    run.check("review frame has visible contrast", contrast_ok, detail=f"rgb standard deviation {std:.1f}")
    run.check("red review body is framed", subject_ok, detail=f"body pixels {subject_pixels}")
    # One complete initial review state at the earliest useful simulated time.
    if exposure_ok and contrast_ok and subject_ok:
        logger.image("camera/front", rgb)
    logger.value("base", rr.Transform3D(parent_frame="World", child_frame="robot/base"))
    logger.value("scene/body", rr.Boxes3D(centers=[[0.0, 0.0, 0.25]], sizes=[[0.4, 0.3, 0.5]], colors=[[230, 51, 26]]))
    logger.scalar("metrics/speed_mps", 0.0)
```

### Panels

The platform opens the blueprint tree, selection inspector, and time panel
hidden, on the automatic layout and on yours alike, so the window belongs to
the telemetry. Give a panel an explicit state to opt it back in:

```python
rrb.Blueprint(rrb.Grid(*views), rrb.SelectionPanel(state="Expanded"))
```

`rrb.PanelState` is `Hidden`, `Collapsed`, or `Expanded`, and the three panels
are `rrb.BlueprintPanel`, `rrb.SelectionPanel`, and `rrb.TimePanel`.

## Troubleshooting

| Symptom | Likely cause and fix |
| --- | --- |
| Stream stays `starting` | Declare the lease before boot: `--stream` on the dispatch, or `antioch jupyter stream --kernel KERNEL_ID --json` before the cell calls `antioch.boot()`. Stop a stale kernel or wait for the current lease holder. |
| Live viewer has no data | `Logger` calls only write during an active scenario or session. Dispatch with `--stream` (or `ScenarioSession(..., live=True)` behind a native/Jupyter lease), then emit at least one sample on a valid relative path. A headless run still records the same samples in its `.rrd`. |
| Playback starts empty, then the scene appears seconds later | The simulator started at 0 s, but the first `Logger` calls came after setup, settling, or a task phase. Log one complete review state on the earliest useful rendered step. Rerun cannot show an entity before its first sample. Disable platform capture if its uncontrolled early frames make that gap misleading. |
| Blueprint not applied | `set_blueprint` must be called while the run is active, and `SpatialInformation` needs the exact `target_frame` your transforms use. Remember an author blueprint replaces the automatic one entirely. |
| One pane visible, the rest behind tabs | The blueprint's root is a bare list of views. Wrap them in `rrb.Grid`/`Horizontal`/`Vertical`. |
| `/antioch/viewport` frames are solid black | Nothing in view is lit. Start with a `DomeLight` near 400 plus a `DistantLight` near 1500, render, then measure the frame. The run warns once when every frame is black. Streaming makes no difference. |
| Frames are white or pale grey | Lights or camera exposure are too high. Reduce intensity before changing exposure. A mean above 220 fails the generic exposure screen; the SDK warns when every platform frame is overexposed. |
| `/antioch/viewport` shows a scene far away, empty, or flat grey | It is an uncontrolled read-back of Kit's active viewport. An authored camera prim does not switch it. After the final reset, aim `/OmniverseKit_Persp`, render, capture, validate, and log the accepted frame at `/viewport`; or use a dedicated render product and disable platform capture. |
| The primary image has useful mean/std but misses the task | Mean and standard deviation only reject degenerate images. Add a content oracle: semantic-mask pixel count, projected subject bounds, or known-colour occupancy. Reject the frame before `Logger.image` when that oracle fails. |
| The 3D pane is empty axes | The recording has transforms but no drawable geometry. Add `Boxes3D`, `Points3D`, `Mesh3D`, or another drawable archetype under the pane's contents path. |
| Playback opens on `wall_time` | Add `rrb.TimePanel(timeline="sim_time")` to an authored blueprint. Automatic blueprints select `sim_time`. |
| Far fewer `/antioch/viewport` frames than the run is long | Frames ride a 0.5 s sim-time cadence, so expect about two per simulated second and no more — a ten-second scenario is twenty frames, not a video. Every run logs what it got (`viewport telemetry captured N frames over X.Xs of simulation, N.N per second`); compare with the cadence before theorising. Materially under two per second is a defect worth reporting. `viewport telemetry stopped after N frames — the renderer returned no further frame in the last X.Xs of simulation` means the renderer stopped answering: count rows with the RRD reader and report it. |
| No `/antioch/viewport` frames at all | Capture was disabled (`capture=False`, `ANTIOCH_TELEMETRY_CAPTURE=0`), the run never stepped a simulator (frames ride the physics callback), or the source failed — the last logs `default viewport capture stopped` once and never fails the run. |
| The `.rrd` is enormous | Something is logging raw images. Route every picture through `Logger.image`, which downsamples and JPEG-compresses; `logger.value(..., rr.Image(...))` stores the frame as-is. |

## Reading a recording back

```bash
antioch scenario download SCENARIO_RUN_ID
rerun rrd stats <file>
rerun rrd print <file> | head
```

On a read-only filesystem the CLI may print `ERROR re_analytics: Failed to
initialize analytics` — upstream telemetry noise, not a verification failure.
Use the statistics that follow; the warning does not change them.
