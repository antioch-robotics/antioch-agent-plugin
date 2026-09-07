---
name: scenario-design
version: "1.2.4"
description: >
  Teaches agents to design Antioch scenarios end to end — the `@antioch.scenario`
  unit and its `ScenarioRun` handle, declaring cases and parameters, modelling
  pass/fail with `run.check`, recording results and artifacts, the platform's
  automatic viewport read-back and viewer layout, `antioch.Logger` for scalars and
  camera images, Rerun blueprints, and how to read a finished run back.
  Use it when writing or reviewing a scenario or suite, when deciding what a run
  should pass or fail on, when a run's dashboard is empty, grey, black, or
  short of frames, when choosing a blueprint or diagnosing a livestream, and when
  verifying an `.rrd`. Not for the CLI's command surface (load
  `antioch-platform`) or for Isaac authoring itself (load `isaac-sim-6` or
  `isaac-lab-3`).
---

# Designing scenarios on Antioch

A scenario is a 3D integration test written as a Python function. Antioch runs
the function in a GPU session and keeps its inputs, pass/fail outcome, results,
logs, telemetry, and artifacts together. A useful scenario gives an engineer
enough evidence to understand what the robot did and why it passed or failed.

The deep Rerun surface — blueprint constructors, entity-path rules, live
streaming, and the troubleshooting table — lives in
[references/telemetry.md](references/telemetry.md). Read this file first;
load that one the moment you are authoring a viewer layout, wiring live
telemetry, choosing 0.36.0 blueprint constructors, or diagnosing a recording
— a dashboard that is empty, grey, black, or short of frames, an `.rrd` that
will not open, or entity paths that never appear.

## Preserve the task

Use the platform skill for asset lookup, dispatch, and result readback.
Respect an existing project asset and the user's requested scope. An
explanation does not authorize a run, and a diagnosis does not authorize
changing thresholds or replacing physical behavior. When execution is
requested, start with a representative case before a costly sweep.

## Research first

Scenario code is raw Isaac + Rerun code. Before writing or debugging any of
it, use the `antioch-research` MCP (see the `antioch-research` skill):
`research_search` across all pinned corpora for APIs, patterns, and errors;
`kind='source'` to localize implementations; `research_open` for whole
files. This skill orients scenario structure — research grounds every API
call you write inside one.

## The unit

```python
import antioch

logger = antioch.Logger("vial")


@antioch.scenario(tags=["vial", "smoke"], restart_services=("autonomy",))
def vial_place(run: antioch.ScenarioRun, seed: int = 1) -> None:
    """Place one vial in the rack and check it seated upright."""
```

- The runner owns Kit startup and one scenario per process. Do not call
  `antioch.start_simulation()` from a scenario body; declare what the simulator must be
  with `@antioch.scenario(config=...)`. A scenario that needs no simulator can
  pass `config=None`; Antioch still saves its results without paying for Kit.
- `run` is always the first positional parameter. Every other parameter needs a
  scalar annotation and a default — that signature *is* the scenario's
  parameter schema.
- Keep simulator imports inside the function body. `pxr`, `omni`, `carb`,
  `isaacsim`, and `isaaclab*` at module scope break discovery on a computer with
  no simulator.
- Native scripts and notebooks start Kit themselves. A source-backed decorated
  scenario can be imported and used as a **programmatic call** after
  `antioch.start_simulation()`; the public callable omits the run and returns
  `run.results`. Inside a managed session command, the call creates a saved
  scenario run. Outside managed compute, results,
  checks, and logging are transient, and `run.add_artifact(...)` is
  unavailable. Keep `antioch.Scenario(...)` as a context manager for
  an undecorated block. Never construct `ScenarioRun` directly.
- Use `profile="perception"` when the scenario needs auxiliary services from
  that `antioch.yaml` profile; the frozen revision includes them.
- `restart_services=("autonomy",)` is background policy only. Antioch restarts
  those named services from the pinned project revision before each background
  child and waits for health. Interactive execution ignores the declaration
  because it is the user's live environment; the user chooses when to run
  `antioch services restart`.

## A well-modelled scenario declares five things

| | | |
| --- | --- | --- |
| Identity | what this run *is* | the function name, `description`, `tags` |
| Inputs | what varies | typed parameters and `antioch.case(...)` |
| Task | what the robot must achieve | the body |
| Verdict | whether it achieved it | `run.check(...)` |
| Evidence | how a reader can tell | results, artifacts, telemetry |

The fourth is the one most scenarios get wrong. A run that only reports
"the code did not crash" is not evidence about a robot.

## Declare the verdict with `run.check`

```text
ScenarioRun.check(criterion: str, passed: bool, *, detail: str = "") -> bool
```

Each call records one named criterion and its verdict, and returns `passed` so
you can branch on it. On normal completion, any final failed check yields `FAILED`; no checks or
all final checks passing yields `PASSED`, unless an explicit outcome overrides them. So the outcome is the
*task's* outcome rather than a proxy for whether the process survived.

```python
tilt_deg, seat_mm = measure(scene)
run.check("upright", tilt_deg <= GATE_TILT_DEG, detail=f"{tilt_deg:.2f}° <= {GATE_TILT_DEG}°")
run.check("seated", seat_mm <= GATE_SEAT_MM, detail=f"{seat_mm:.2f} mm <= {GATE_SEAT_MM} mm")
run.check("at rest", speed_mps <= GATE_REST_MPS, detail=f"{speed_mps:.4f} m/s")
```

Rules that make checks useful:

- **One check per criterion the task actually defines.** Four gates, four
  checks — not one `assert` over their conjunction. A chained `assert` stops at
  the first failure and hides the other three measurements, which is exactly
  the information you opened the run to see.
- **Always pass `detail` with the measurement**, not a restatement of the
  name. `tilt 7.31° > 5.00°` is a finding; `upright failed` is not.
- **Checks do not stop the run.** Keep measuring. Re-checking the same
  criterion replaces its verdict in place. For a condition that must hold
  throughout motion, accumulate failures or a worst-case measurement and
  emit the final criterion. A later passing sample must not erase a drop,
  collision, or earlier limit violation.
- **`assert` and `run.fail(reason)` are for "cannot continue"** — the rig did
  not build, the policy file is missing. Both stop the body immediately and
  mark the run `FAILED`.
- **`run.skip(reason)` is for an unmet precondition**, not for a failure.
- **`run.set_outcome(...)` overrides the checks**, for a scenario whose
  judgement is more subtle than a conjunction. An unexpected exception overrides both and reports `ERRORED`;
  assertion failures and `run.fail` instead produce `FAILED`.

Checks land in `results` under the reserved `checks` key. `antioch scenario
show` displays them in a dedicated checks row. Do not write that key yourself.

## Parameterize with cases

Cases turn one authored scenario into many runs that share a name, so a suite
compares like with like:

```python
@antioch.scenario(
    tags=["vial"],
    cases=[
        antioch.case(id="nominal", tags=["smoke"]),
        antioch.case({"friction": 0.15}, id="slippery", tags=["edge"]),
        antioch.case(grid={"seed": range(50)}, id="seed-{seed}"),
    ],
)
def vial_place(run: antioch.ScenarioRun, seed: int = 1, friction: float = 0.6) -> None: ...
```

`antioch.case(params, *, grid=..., combinations=..., id=..., tags=...)` gives a
singleton, a Cartesian grid, or correlated rows. `id` is a `str.format`
template over the resolved parameters; omit it for stable derived ids. Tag
cases so a suite can select them (`smoke` for the fast path, `cert` for the
long one), and declare the suites themselves under `suites:` in
`antioch.yaml` — the `antioch-platform` skill owns the selector shape.

Use separate cases for independent outcomes, retries, or comparisons. Use
an internal loop when the iterations form one evaluation or dataset with a
shared lifecycle. Record per-iteration detail when aggregate results would
hide failures. Review the expanded case count before submitting it.

## Record the evidence

```text
ScenarioRun.add_result(name: str, value: Any) -> None
ScenarioRun.add_results(results: dict[str, Any]) -> None
ScenarioRun.add_artifact(path: str | Path, *, name: str | None = None, content_type: str | None = None, description: str | None = None) -> None
```

- Results must be valid JSON and fit in the saved scenario results. Put the summary
  there, such as thresholds, counts, and aggregate error, and save a
  per-episode table as an artifact.
- Artifacts upload directly from the service to object storage. Any media type
  is fine. Give each one a one-line `description` (at most 140 characters) saying
  what the file is — the console's download menu and `antioch scenario show`
  display it beside the name:
  `run.add_artifact("outputs/summary.json", description="Per-case contact summary")`.
  Two names are reserved with pinned shapes: `telemetry` is the session's own
  `.rrd` recording (`application/vnd.rerun.rrd`), and `output` is the runner's
  captured stdout/stderr (`output.log`, `text/plain; charset=utf-8`, at most
  4 MiB).
- Record the thresholds a check used, not just its verdict. A reader six weeks
  later needs to know what "passed" meant.

## Default telemetry

Antioch creates a recording and derives a layout from the data that arrives.
A run with no samples does not automatically have a useful dashboard:

- **A diagnostic read-back of Kit's active viewport.** Once physics is
  stepping, the platform logs one JPEG to `/antioch/viewport` about twice a
  simulated second, 640 px wide, capped at 600 frames. It does not select,
  aim, or focus a camera. A USD camera at `/World/Camera` is not automatically
  the active viewport camera. Never use this uncontrolled picture as the only
  visual evidence for a bench-scale task.
- **A viewer layout derived from what was logged** — a 2D view per entity
  carrying an image or video, authored images first and
  `/antioch/viewport` last, one time-series view per scalar path, and a 3D
  view only when a drawable 3D archetype exists. `Transform3D` positions
  geometry but draws no mesh, box, or point by itself.
- **`sim_time` as the default viewer timeline.** The stored time panel names
  it explicitly, so playback does not open on `wall_time`.
- **The viewer's control panels collapsed by default**: the blueprint tree,
  selection inspector, and time panel do not take space from telemetry, and
  the collapsed time panel still shows the narrow timeline under the
  viewports. Give a panel an explicit state only when the reviewer needs it.

Capture rides the physics-step callback, never changes the run's outcome, and
reports what it got at the end (`viewport telemetry captured N frames over
X.Xs of simulation, N.N per second`). It warns when every frame is black,
underexposed, overexposed, or nearly uniform. Those warnings diagnose the
active viewport; they do not replace a scenario check. Turn platform capture
off when a dedicated camera is the complete visual record, or when its cost is
not justified:
`@antioch.scenario(capture=False)`, `Scenario(..., capture=False)`, or
`ANTIOCH_TELEMETRY_CAPTURE=0`.

Rerun shows explicit samples, not the USD stage. Each `Logger` call writes one
value at the current `sim_time` when simulation time is available; it does not discover scene objects, preserve
an unlogged state, or backfill earlier time. If the first camera and drawable
3D samples arrive at 3 s, those panes are empty before 3 s even though the
assets already exist in Kit.

When visual review is required, emit an initial useful state after reset and
camera setup: the authored image, any required drawable scene geometry, and
baseline metrics. Then log again when the evidence changes. The scenario
decides those moments; there is no required step count or telemetry cadence.
If platform capture would record the uncontrolled camera before the owned
camera is ready, aim it before the first rendered step or set `capture=False`.

## Own the review camera

Choose a view that shows the task. For a classic World viewport, aim the
actual active camera after reset, render, then read back. A named USD camera
does not automatically select itself as the viewport. A dedicated render
product is useful when calibration or independence from the livestream matters.

Check decoded shape/range, then task content: projected subject bounds,
semantic-mask pixels, or another feature that proves the subject is visible.
Mean and variance alone cannot distinguish the right scene from a well-lit
empty one. Dark or low-contrast scenes may be intentional; no fixed exposure
band is correct for every task.

Keep diagnostic frames when checks fail. Log the image and the failed
measurement; do not hide the evidence by publishing only accepted frames.
See the Isaac Sim sensor/rendering references for capture ownership and units.

## Log your own telemetry

One logger at module scope, reused; it holds a prefix and resolves the active
run on every call, so the same helper works in a scenario and in an ordinary
script.

```text
antioch.Logger(prefix: str = "") -> Logger
Logger.scalar(path: str, value: float) -> None
Logger.image(path: str, pixels: Any, *, max_width: int = 960, jpeg_quality: int = 65) -> None
Logger.value(path: str, obj: Any) -> None
Logger.debug/info/warning/error(message: str, *, path: str = "logs") -> None
```

Pass a CPU-accessible RGB or RGBA array with shape `(height, width, 3 or 4)`
to `Logger.image`, preferably `uint8` in `[0, 255]`. Alpha is discarded.
Other dtypes are clipped and cast, not rescaled: convert known `[0, 1]` color
samples to `[0, 255]` before logging, or they become nearly black. Copy GPU
buffers to CPU explicitly. Keep raw depth, masks, and other measurement data
in lossless artifacts; a colorized review image is a separate representation.

```python
logger.scalar("metrics/tilt_deg", tilt_deg)
logger.image("camera/bench", rgb)  # camera frames go here
logger.value("metrics", {"error": error, "reward": reward})
```

**Log pictures through `Logger.image`, never `logger.value(..., rr.Image(...))`.**
`image` downsamples and JPEG-compresses on the way in, the same compression
the platform's own capture uses; a few hundred raw frames is the difference
between an RRD somebody opens and one they give up downloading.

Sample cameras at a rate a person would watch — a handful of frames per
simulated second — and prefer more frames at a smaller size over a few
enormous ones. Choose size and compression from the evidence needed, and measure the resulting file.

Antioch stamps every write with `wall_time`, and with `sim_time` once Kit is
running. One call is one sample at that current time; later calls do not fill
an earlier gap. Stored blueprints default playback to `sim_time`. Within an Antioch-managed recording, do not replace its sinks or clock with
`rr.init`, global connection/time calls, or a separate blueprint stream.
Use `run.set_blueprint` for the active recording.

## Choose a layout, or don't

The automatic blueprint leads with authored cameras, puts platform viewport
capture last, selects `sim_time`, and collapses all control panels. Author a
layout only when the evidence needs a specific composition.

```python
run.set_blueprint(rrb.Blueprint(rrb.Horizontal(rrb.Grid(*views), rrb.Grid(*series))))
```

An author blueprint replaces the automatic one **entirely** — include a view
for every entity you still want visible. Include `/antioch/viewport` only when
that diagnostic view helps. Name a container (`Grid`, `Horizontal`,
`Vertical`) as the root: a bare list of views serializes as a tab container,
which shows one pane and hides the rest. Omit control panels unless the review
needs one; the SDK supplies a collapsed time panel on `sim_time`, which keeps
the timeline under the viewports. Keep the time panel collapsed when a reviewer needs the scrubber;
hiding it removes that control.
[references/telemetry.md](references/telemetry.md) owns the verified 0.36.0
constructors, `SpatialInformation`, and the live-versus-recorded flow.

## Watch it live

Attached `antioch scenario run` and `antioch suite run` request the session's
livestream by default and share `--stream/--no-stream`; a detached run is
headless. Each scenario reserves the session's livestream while its simulation
process runs, and the attached command shows progress until the run finishes.
Inspect the recorded telemetry and artifacts after completion. A native script
under `antioch services exec` and a Jupyter kernel use the same single
livestream slot; `services exec` takes the same `--stream/--no-stream` pair,
and a kernel must be assigned the stream before simulator startup. There is no
stream size or rate field in simulation code or `antioch.yaml`.

## Run it and read it back

```bash
antioch scenario collect                                   # discovery and schema errors, locally
antioch scenario run --scenario vial_place                  # one scenario, attached
antioch suite run smoke                                    # a declared suite
antioch scenario show SCENARIO_RUN_ID                      # verdict, checks, results, artifacts
antioch scenario show SCENARIO_RUN_ID --logs               # captured output
antioch scenario download SCENARIO_RUN_ID                  # the .rrd and every artifact
```

After a requested run, inspect the evidence needed for that task. Changing
checks alone does not require adding a camera, 3D geometry, or a viewer step
to a scalar-only or simulator-free evaluation:

1. `antioch scenario show SCENARIO_RUN_ID` — is the outcome the task's
   outcome, and does every criterion you meant to declare appear?
2. Download required artifacts with `antioch scenario download SCENARIO_RUN_ID`.
   For recorded telemetry, use `rerun rrd stats <file>` and
   `rerun rrd print <file> | head` to inspect the expected entity paths and
   sample timestamps. Simulation samples need `sim_time`; simulator-free
   measurements use their available timeline.
3. When images are evidence, decode them and apply the task-specific content oracle.
   Check timestamps and retained failure samples. Nonzero mean is not proof.
4. When visual layout or playback is in scope, open the recording in the viewer.
   Check the required panes and timeline. An authored camera should lead when
   it is the primary evidence; a requested 3D pane must draw geometry rather
   than empty axes. Report any viewer check that could not run.

Native, scenario, and suite execution all use sessions. Attached scenario execution is serial on the selected live session;
ordinary headless service commands can run alongside it. Background work reuses revision-pinned
background sessions. Scenario and suite records keep outcomes and evidence
independently and never own session compute or usage.

Deeper read-back — filtering run history, per-service logs, artifact keys,
deletion — is the `antioch-platform` skill's scenarios reference; suite
follow-up and cancellation is its suites reference.

## Checklist

Apply camera, 3D, and viewer checks only to the visual evidence the task needs.
Scalar-only and simulator-free evaluations still need measured criteria and
result readback, not an invented visual workload.

- [ ] Every criterion the task defines is a `run.check` with a measured `detail`.
- [ ] `assert` / `run.fail` appear only where the run genuinely cannot continue.
- [ ] Cases represent independent outcomes; internal loops retain required detail.
- [ ] Thresholds are in `results`; per-episode detail is an artifact.
- [ ] Camera frames go through `Logger.image`.
- [ ] The review camera is aimed after reset, rendered, decoded, and validated
      before its frame is accepted.
- [ ] A task-specific oracle proves the subject is in frame; mean/std only
      screen exposure and flat captures.
- [ ] Entity paths are a stable hierarchy a blueprint can select without
      knowing per-run values.
- [ ] The earliest useful rendered step logs one complete camera, drawable 3D,
      and metric state; setup and settle time do not leave an unexplained gap.
- [ ] A 3D view has a drawable archetype such as `Boxes3D`, `Points3D`, or a
      mesh; transforms alone are not visible.
- [ ] Lighting and image checks match the task; failure frames remain available.
- [ ] The blueprint timeline matches the recorded measurements; required viewer
      checks ran, or are reported as unrun.
