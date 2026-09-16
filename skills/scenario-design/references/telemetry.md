# Telemetry and viewer layouts

The SDK and viewer use **Rerun 0.36.0**. Ground additional constructors in
that pin. Antioch owns recording sinks and clocks; do not call `rr.init`
or replace them with global connection/time operations.

## Log evidence

```python
logger = antioch.Logger("robot")
logger.scalar("metrics/error_m", error_m)
logger.image("camera/front", rgb)
logger.value("state", {"phase": phase})
```

Paths are relative, nonempty segments using alphanumerics, `_`, `.`, or
`-`, separated by `/`. Prefixes are prepended; blueprint selectors use
absolute paths such as `/robot/camera/front`.

`Logger.image` accepts CPU RGB/RGBA arrays, preferably `uint8` in
[0,255]. It drops alpha, downsamples, and JPEG-encodes. Other dtypes are
clipped/cast, not rescaled: convert [0,1] colors explicitly. The first image
fixes that entity's canvas; later frames fit without enlargement, stretching,
or cropping. Use separate paths for separate camera canvases.

Already-encoded JPEGs use
`logger.value(path, rr.EncodedImage(contents=jpeg_bytes, media_type="image/jpeg"))`.
Metric depth uses `rr.DepthImage(depth_m, meter=1.0)` through `value`.
Keep lossless evaluation arrays in artifacts; review images are compressed.

Each call samples the current `wall_time` and available `sim_time`.
Nothing backfills earlier state or discovers USD geometry. Log initial,
important transition, and terminal evidence; sample motion at a useful cadence.
Keep failure frames and monitor finalized recording size.

## Automatic viewport and layout

Automatic capture logs the active viewport at about 2 frames per simulated
second, 640 pixels wide, capped at 600 frames. It follows physics callbacks,
does not select a camera, and reports skipped/failed captures. Warnings do not
change the scenario verdict. `capture=False` disables this sampler, not
authored telemetry or the file sink.

The default layout derives image panes, scalar plots, and drawable 3D views
from logged data and selects `sim_time`. A `Transform3D` places geometry
but draws nothing; log boxes, points, or meshes for a visible 3D scene.
Check coordinate-frame IDs and quaternion conventions at this boundary.

## Custom layouts

```python
import rerun.blueprint as rrb

run.set_blueprint(
    rrb.Blueprint(
        rrb.Horizontal(rrb.Spatial2DView(origin="/robot/camera/front", name="Front"), rrb.TimeSeriesView(origin="/robot/metrics", contents="/robot/metrics/**"))
    )
)
```

Set the blueprint while the run is active. It replaces the automatic layout,
so include every required entity. Explicit containers show simultaneous panes;
bare views become tabs. `rrb.SpatialInformation` requires a `target_frame`
matching the logged frame graph. Default collapsed panels preserve the scrubber.

## Live and saved evidence

Saved RRDs publish as the reserved `telemetry` artifact. Offline scopes do
not upload; [caller recording](recording.md) owns that boundary.
Live telemetry separately requires the process's session stream grant.
`livestream=None` inherits it; `True` cannot manufacture one.
`run.live_uri` is `None` without a live sink.

Download the run and read its file with pinned `rerun rrd stats`,
`rerun rrd print`, or `rerun.experimental.RrdReader`. Older dataframe
APIs/DataFusion are not part of the environment. Check entity paths and sample
times, decode relevant images, and open the layout when visual review is in
scope. Empty panes often mean missing samples/drawables, wrong selectors or
frames, or a different timeline—not missing scene geometry in Isaac.
