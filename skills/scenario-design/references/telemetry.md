# Telemetry and viewer layouts

The SDK and viewer use **Rerun 0.38.1**; ground additional constructors in
that pin. A scenario or an [`antioch.Telemetry`](#scripts-and-notebooks) scope
opens one native `rr.RecordingStream` per process and owns its file and live
sinks. While it is open it is Rerun's global default, so `rr.log`,
`rr.set_time`, archetypes, and `rr.send_blueprint` land in it; pass
`recording=run.recording` or `recording=telemetry.recording` to route
explicitly. Do not call `rr.init`, `rr.disconnect`, or `rr.serve_grpc`: they
replace or detach the recording's sinks.

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
It drops alpha, downsamples, and JPEG-encodes before returning. Encode cost
grows with the input, so log
images at the size you want recorded, from a fixed-size render product rather
than the streamed viewport. Other dtypes are clipped and
cast, not rescaled: convert [0,1] colors explicitly. The first image fixes
that entity's canvas; later frames fit without enlargement, stretching, or
cropping, so use separate paths for separate camera canvases.

Already-encoded JPEGs use
`logger.value(path, rr.EncodedImage(contents=jpeg_bytes, media_type="image/jpeg"))`.
Metric depth uses `rr.DepthImage(depth_m, meter=1.0)` through `value`. Keep
lossless evaluation arrays in artifacts; review images are compressed.

Each call samples the current `wall_time` and available `sim_time`. Nothing backfills earlier state or discovers USD geometry. Log at a rate that resolves the event being measured; step simulation separately. Each scalar is a Rerun write on the calling thread, roughly 10–40 µs, so a frame logging dozens of values spends a millisecond or more on them. Scalar/image/value calls outside an active recording do not create a run.

For text, `logger.debug`, `info`, `warning`, and `error` write to process output and also to an active recording. Debug/info use stdout; warnings/errors use stderr. `path=` chooses the text entity, and the logger prefix still applies. These methods keep their process output when nothing is recording. `logger.value` accepts native Rerun archetypes; a numeric mapping becomes scalar channels beneath its path. Use [research](../../antioch-research/SKILL.md) before adding a version-sensitive archetype or blueprint constructor.

## Simulation time without Kit

Kit runs stamp `sim_time` from physics steps on their own. A loop that steps
Newton, MuJoCo-Warp, or another simulator directly, such as a `config=None`
scenario, has only `wall_time` until it sets the clock: call
`run.set_sim_time(seconds)` after each step with the simulated time since the
simulation began, then log.

```python
sim_s = 0.0
for _ in range(steps):
    solver.step(state_0, state_1, control, contacts, frame_dt)
    state_0, state_1 = state_1, state_0
    sim_s += frame_dt
    run.set_sim_time(sim_s)
    logger.scalar("metrics/error_m", error_m)
```

Later samples carry `sim_time`, `run.sim_s` reports it, and replay runs at
simulated speed. Pass `0.0` after a reset to start the timeline over. Do not call `rr.set_time` for this. When Kit's
physics clock already drives the run, `set_sim_time` raises `StateError`.

## Default layout

Scenarios record only what the code logs; nothing captures the viewport on
its own. Without a blueprint, Antioch stores none and the Rerun viewer lays
the recording out itself. Recordings over 256 MiB draw a size warning. A
`Transform3D` places geometry but draws nothing; log boxes, points, or meshes
for a visible 3D scene.

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

Set the blueprint while the run is active. It replaces the viewer's automatic
layout, so include every required entity and select the timeline holding its
samples. `run.set_blueprint`, `blueprint=` on the scenario or `Telemetry`, and
`rr.send_blueprint` all send the layout once, exactly as given; nothing
replaces it at finalize.
Explicit containers show simultaneous panes; bare views become tabs.
`rrb.SpatialInformation` requires a `target_frame` matching the logged frame
graph.

## Scripts and notebooks

`antioch.Telemetry` records the same way without a scenario: it saves an RRD
and streams to Mission Control's **Telemetry** pane, with no run record,
check, or upload.

```python
import antioch
import rerun as rr

with antioch.Telemetry("tuning", rrd_path="tuning.rrd") as telemetry:
    for step in range(100):
        rr.set_time("step", sequence=step)
        rr.log("metrics/error", rr.Scalars(error_at(step)))
```

In a notebook, call `telemetry = antioch.Telemetry("tuning").start()` in one
cell and `telemetry.stop()` in a later one; both are safe to repeat, and a
stopped handle does not restart. Without `rrd_path`, `telemetry.rrd_path`
names a temporary file; copy it out before releasing the session.
`live=False` keeps only the file. One recording is active per process, so stop telemetry
before starting a scenario in the same kernel. The file is sealed when the
scope closes or the process exits normally, including Ctrl-C; a stop signal
such as `SIGTERM` ends a plain script without sealing it. Stop telemetry
before closing a simulator yourself: Kit's close ends the process first.

## Live and saved evidence

Saved scenario RRDs publish as the reserved `telemetry` artifact. Offline
scopes do not upload; [in-session recording](recording.md) owns that
boundary. Every scenario and live `Telemetry` scope in a session publishes on
port 9876, with or without `--stream`, and anyone on the team can watch a
dispatched run on its console page. The port belongs to the latest producer:
a process that has finished recording hands it over, while one that starts
beside a process still recording, or beside a foreign listener such as
`rr.serve_grpc`, keeps only its file and warns. `run.live_uri` is `None`
without a live sink. Native video follows `SimulationConfig(stream=...)`, off
by default; a `--stream` or `--no-stream` on `scenario run`, `suite run`,
their reruns, `service exec`, or `jupyter cell` overrides it. Neither affects
telemetry.

Download the run, or only its file with
`antioch scenario download SCENARIO_RUN_ID --artifact telemetry`, and read it with pinned `rerun rrd stats`,
`rerun rrd print`, or `rerun.chunk.RrdReader`; older dataframe
APIs/DataFusion are not part of the environment. Empty panes usually mean
missing samples/drawables, wrong selectors or frames, or a different
timeline, not missing scene geometry in Isaac.

Use [navigation and capture](../../agentic-simulation/references/viewport.md) to aim the viewport, [native cameras](../../isaac-sim-6/references/isaac-camera.md) for measured camera data, and [run history](../../antioch-platform/references/scenarios.md#find-and-analyze) to download saved evidence. Return to [scenario design](../SKILL.md#choose-the-next-reference) for checks, results, and recording boundaries.
