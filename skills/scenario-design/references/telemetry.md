# Rerun telemetry and blueprints

Use this for viewer layouts, live telemetry, and recording diagnosis.
The SDK and web viewer are pinned together at **Rerun 0.36.0**.
Ground constructor and frame semantics in that release before adding layouts.

## Recording versus live transport

A managed Antioch scenario finalizes its RRD and uploads the reserved
`telemetry` artifact. A native `Scenario` also records to a file; outside
managed compute that file is local output, not automatically a durable
platform artifact.

A live sink is separate. The process must own the session's stream grant
before simulator startup. A stream-available badge is capacity, not evidence
that a scenario is already producing data.

In a native script or notebook that already started Kit:

```python
import antioch

with antioch.Scenario("inspect", rrd_path="inspect.rrd") as run:
    logger = antioch.Logger("robot")
    logger.scalar("metrics/error", 0.0)
```

The public constructor option is `livestream`, not `live`. Its default
`None` follows the managed process grant and resolves to false outside
Antioch. Explicit `livestream=True` cannot manufacture a stream grant.
`run.live_uri` is `None` when no live sink is serving.
`capture=False` disables automatic viewport readback for this recording,
not the file sink or authored telemetry.

Do not construct `ScenarioRun` directly. Do not replace Antioch's sinks,
global clock, or recording with separate Rerun initialization calls.

## Entity paths and time

`Logger` paths are relative: alphanumeric segments plus `_`, `.`, or `-`,
separated by `/`, with no leading/trailing slash or empty segment.
The logger prefix is prepended. Blueprint selectors use absolute paths.

```python
logger = antioch.Logger("robot")
logger.scalar("metrics/speed_mps", speed)
logger.image("camera/front", rgb)
```

These produce `/robot/metrics/speed_mps` and `/robot/camera/front`.
Use `Logger.image` for raw pixels: it downsamples and JPEG-compresses them
for bounded visual review, and re-encodes whatever it is given. A frame a CV
service already encoded goes through
`logger.value(path, rr.EncodedImage(contents=jpeg_bytes, media_type="image/jpeg"))`
unchanged, and metric depth through
`logger.value(path, rr.DepthImage(depth_m, meter=1.0))`; the default layout
gives each a 2D view. Raw tensors or lossless data needed by an evaluation
belong in a separate artifact.

A logging call writes a sample now; it does not backfill earlier time or
discover geometry from USD. The recording has `wall_time` and adds
`sim_time` when simulation time is available. Do not claim time coverage
before the first sample or after logging stopped.

Log an initial useful review state after setup, then sample at a cadence that
captures relevant motion, and log each state change and the terminal state
when it happens, whatever the cadence: a periodic-only sampler can miss the
final `done`. Explain setup gaps. Keep diagnostic failed frames with their
checks instead of deleting them from the record.

## Geometry and frames

A `Transform3D` places entities; it does not draw a robot. A 3D view needs
drawable data such as `Boxes3D`, `Points3D`, or `Mesh3D`.
Check coordinate frame IDs and quaternion convention at the Rerun boundary.

`rrb.SpatialInformation` requires `target_frame`. It must match the frame
graph used by the logged transforms, not a guessed stage path. Inspect the
recorded frame IDs when a view is empty or displaced.

## Layouts

Antioch can derive a layout from logged images, scalar series, and drawable
3D data. Author a blueprint only when the review needs a specific arrangement.

```python
import rerun.blueprint as rrb

run.set_blueprint(
    rrb.Blueprint(
        rrb.Horizontal(
            rrb.Spatial2DView(origin="/robot/camera/front", name="Front camera"),
            rrb.TimeSeriesView(origin="/robot/metrics", contents="/robot/metrics/**", name="Metrics"),
        )
    )
)
```

An authored layout replaces the automatic layout. Include each entity that
must remain visible, including `/antioch/viewport` if that diagnostic view is
useful. Use an explicit container such as Grid/Horizontal/Vertical when the
review needs simultaneous panes; a bare set of views becomes tabs.

The SDK defaults control panels to collapsed and selects `sim_time`.
A collapsed time panel preserves the scrubber. Set another timeline or panel
state only for a deliberate review need, such as process latency on
`wall_time`. `run.set_blueprint` must be called while the run is active.

## Diagnose the recording

| Symptom | Check |
|---|---|
| Live viewer empty | Process grant, active Scenario, live sink, and actual Logger calls |
| Playback begins empty | First sample times, setup interval, and whether drawable geometry was logged |
| Image looks wrong | Decode it; check camera, synchronization, subject, and task-specific image criteria |
| Only one pane visible | Tab root versus explicit layout container |
| 3D axes but no subject | Drawable archetype and contents selector, then transform frames |
| Blueprint missing | Active-run setter, entity paths, replacement semantics, and frame IDs |
| No viewport samples | Capture disabled, no physics steps, frame limit, or capture failure logs |
| Fewer frames than expected | Simulated duration, the skipped ticks and cap named on the capture line, render readiness and timestamps |
| RRD too large | The finalize size line and its rate per second, raw image logging, oversized geometry, or unnecessary sample rate |

Automatic viewport capture is a bounded diagnostic sampler, not video and
not an outcome oracle. Warnings about black, overexposed, or stopped capture
do not fail the scenario. Use a named task check when visual content matters.

After a requested run, use `antioch scenario show SCENARIO_RUN_ID` and
`antioch scenario download SCENARIO_RUN_ID`. Read the file back with the
pinned `rerun-sdk==0.36.0` the SDK installs: `rerun rrd stats <file>`,
`rerun rrd print <file>`, and `rerun.experimental.RrdReader` from Python.
The dataframe API of older Rerun releases and the DataFusion extra are not
part of the pinned environment. Inspect statistics and sample timestamps,
decode the primary image, and open the layout when visual review is in
scope. Report what was checked, including any viewer or runtime step that
could not run.
