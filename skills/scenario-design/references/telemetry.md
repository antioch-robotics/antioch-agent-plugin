# Rerun telemetry and blueprints

Context: read this when authoring a viewer layout, wiring live telemetry, or
diagnosing a recording that does not look right. `SKILL.md` owns the scenario
model and the telemetry defaults; this file owns the Rerun surface underneath.

The SDK and browser viewer are pinned together at `rerun-sdk==0.36.0` and
`@rerun-io/web-viewer==0.36.0`. Every constructor below is verified in that
release.

## Live versus recorded

Every `ScenarioSession` opens a file sink. A managed scenario finalizes that
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
            rrb.Spatial2DView(origin="/viewport", name="Scene"),
            rrb.Spatial2DView(origin="/robot/camera/front", name="Bench camera"),
            rrb.Spatial3DView(
                origin="/robot", contents="/robot/**", name="Robot", spatial_information=rrb.SpatialInformation(target_frame="World", show_axes=True)
            ),
        ),
        rrb.TimeSeriesView(origin="/robot/metrics", contents="/robot/metrics/**", name="Metrics"),
    )
)


@antioch.scenario()
def robot_dashboard(run: antioch.ScenarioRun) -> None:
    import rerun as rr

    run.set_blueprint(ROBOT_DASHBOARD)
    logger = antioch.Logger("robot")
    logger.value("base", rr.Transform3D(parent_frame="World", child_frame="robot/base"))
    logger.scalar("metrics/speed_mps", 0.0)
```

### Panels

The platform opens the blueprint tree and selection inspector hidden and the
time panel collapsed, on the automatic layout and on yours alike, so the window
belongs to the telemetry. Naming a panel keeps yours:

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
| Blueprint not applied | `set_blueprint` must be called while the run is active, and `SpatialInformation` needs the exact `target_frame` your transforms use. Remember an author blueprint replaces the automatic one entirely. |
| One pane visible, the rest behind tabs | The blueprint's root is a bare list of views. Wrap them in `rrb.Grid`/`Horizontal`/`Vertical`. |
| `/antioch/viewport` frames are solid black | Nothing in view is lit — usually the stage has no light at all. Add one (`create_prim("/World/Light", "DomeLight", attributes={"inputs:intensity": 3000.0})` on Isaac Sim, `sim_utils.DomeLightCfg(intensity=3000.0)` on Isaac Lab). The run says so once at the end: `viewport telemetry captured N frames and every pixel of every one was black`. Streaming makes no difference. |
| `/antioch/viewport` shows a scene far away, empty, or flat grey | Expected on a bench-scale workcell: the platform reads back Kit's active viewport and does not frame the shot, and a headless boot often draws that texture as one flat colour. This is a known limitation, not a bug in your scenario. Log your own camera through `Logger.image` at `/viewport` and give it a view — that is the reliable path, and every shipped example takes it. |
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
The stats that follow are authoritative.
