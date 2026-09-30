# Telemetry and viewer layouts

The SDK and viewer use **Rerun 0.38.1**; ground additional constructors in
that pin. Antioch owns recording sinks and clocks: do not call `rr.init` or
replace them with global connection/time operations.

## Log evidence

```python
logger = antioch.Logger("robot")
logger.scalar("metrics/error_m", error_m)
logger.image("camera/front", rgb)
logger.value("state", {"phase": phase})
```

Paths are relative, nonempty segments that start with an alphanumeric and
then use alphanumerics, `_`, `.`, or `-`, separated by `/`. Prefixes are prepended; blueprint selectors use
absolute paths such as `/robot/camera/front`.

`Logger.image` accepts CPU RGB/RGBA arrays, preferably `uint8` in [0,255].
It drops alpha, downsamples, and JPEG-encodes. Other dtypes are clipped and
cast, not rescaled: convert [0,1] colors explicitly. The first image fixes
that entity's canvas; later frames fit without enlargement, stretching, or
cropping, so use separate paths for separate camera canvases.

Already-encoded JPEGs use
`logger.value(path, rr.EncodedImage(contents=jpeg_bytes, media_type="image/jpeg"))`.
Metric depth uses `rr.DepthImage(depth_m, meter=1.0)` through `value`. Keep
lossless evaluation arrays in artifacts; review images are compressed.

Each call samples the current `wall_time` and available `sim_time`. Nothing backfills earlier state or discovers USD geometry. Log at a rate that resolves the event being measured; step simulation separately. Scalar/image/value calls outside an active recording do not create a run.

For text, `logger.debug`, `info`, `warning`, and `error` write to process output and also to an active recording. Debug/info use stdout; warnings/errors use stderr. `path=` chooses the text entity, and the logger prefix still applies. These methods keep their process output when no scenario is active. `logger.value` accepts native Rerun archetypes; a numeric mapping becomes scalar channels beneath its path. Use [research](../../antioch-research/SKILL.md) before adding a version-sensitive archetype or blueprint constructor.

## Automatic viewport and layout

Automatic capture logs the active viewport at about 2 frames per simulated
second, 640 pixels wide, capped at 600 frames. It follows physics callbacks,
does not select a camera, and reports skipped/failed captures; warnings do
not change the verdict. `capture=False` disables this sampler, not authored telemetry or the file sink. `ANTIOCH_TELEMETRY_CAPTURE=0` disables automatic capture for the process. Neither option disables custom sensor logging.

The default layout derives image panes, scalar plots, and drawable 3D views
from logged data and selects `sim_time` when a simulator ran, else
`wall_time`; recordings over 256 MiB get no derived layout and a size
warning. A `Transform3D` places geometry but
draws nothing; log boxes, points, or meshes for a visible 3D scene.

## Custom layouts

```python
import rerun.blueprint as rrb

run.set_blueprint(
    rrb.Blueprint(
        rrb.Horizontal(
            rrb.Spatial2DView(origin="/robot/camera/front", name="Front"), rrb.TimeSeriesView(origin="/robot/metrics", contents="/robot/metrics/**")
        ),
        rrb.TimePanel(timeline="sim_time"),
    )
)
```

Set the blueprint while the run is active. It replaces the automatic layout,
so include every required entity and select the timeline holding its samples.
Automatic capture logs at `/antioch/viewport`; include it to keep the
capture visible. Send layouts with `run.set_blueprint`, not
`rr.send_blueprint`, which targets Rerun's global stream instead of the
run's recording.
Explicit containers show simultaneous panes; bare views become tabs.
`rrb.SpatialInformation` requires a `target_frame` matching the logged frame
graph.

## Live and saved evidence

Saved RRDs publish as the reserved `telemetry` artifact. Offline scopes do
not upload; [in-session recording](recording.md) owns that boundary. Every
scenario in a session publishes live telemetry on port 9876, with or without
`--stream`, and anyone on the team can watch a dispatched run on its console
page. The port belongs to the latest scenario: a process whose scenario has
finished hands it over, while a scenario that starts beside one still
recording, or beside a non-scenario listener such as `rr.serve_grpc`, keeps
only its file and warns. `run.live_uri` is `None` without a live sink.
`livestream` and `--stream` on `scenario run`, `suite run`, their reruns,
`service exec`, or `jupyter cell` govern native video; `SimulationConfig(stream=False)`
suppresses video only.

Download the run, or only its file with
`antioch scenario download SCENARIO_RUN_ID --artifact telemetry`, and read it with pinned `rerun rrd stats`,
`rerun rrd print`, or `rerun.chunk.RrdReader`; older dataframe
APIs/DataFusion are not part of the environment. Empty panes usually mean
missing samples/drawables, wrong selectors or frames, or a different
timeline, not missing scene geometry in Isaac.

Use [navigation and capture](../../agentic-simulation/references/viewport.md) to aim the viewport, [native cameras](../../isaac-sim-6/references/isaac-camera.md) for measured camera data, and [run history](../../antioch-platform/references/scenarios.md#find-and-analyze) to download saved evidence. Return to [scenario design](../SKILL.md#choose-the-next-reference) for checks, results, and recording boundaries.
