---
name: scenario-design
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
the function on a GPU machine and keeps its inputs, pass/fail outcome, results,
logs, telemetry, and artifacts together. A useful scenario gives an engineer
enough evidence to understand what the robot did and why it passed or failed.

The deep Rerun surface — blueprint constructors, entity-path rules, live
streaming, and the troubleshooting table — lives in
[references/telemetry.md](references/telemetry.md). Read this file first;
load that one the moment you are authoring a viewer layout, wiring live
telemetry, choosing 0.36.0 blueprint constructors, or diagnosing a recording
— a dashboard that is empty, grey, black, or short of frames, an `.rrd` that
will not open, or entity paths that never appear.

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


@antioch.scenario(tags=["vial", "smoke"])
def vial_place(run: antioch.ScenarioRun, seed: int = 1) -> None:
    """Place one vial in the rack and check it seated upright."""
```

- The runner owns Kit boot and one scenario per process. Do not call
  `antioch.boot()` from a scenario body; declare what the simulator must be
  with `@antioch.scenario(sim=...)`. A scenario that needs no simulator can
  pass `sim=None`; Antioch still saves its results without paying for Kit.
- `run` is always the first positional parameter. Every other parameter needs a
  scalar annotation and a default — that signature *is* the scenario's
  parameter schema.
- Keep simulator imports inside the function body. `pxr`, `omni`, `carb`,
  `isaacsim`, and `isaaclab*` at module scope break discovery on a machine with
  no simulator.
- Native scripts and notebooks own their own boot and use
  `antioch.ScenarioSession(...)` as a context manager instead. Never construct
  `ScenarioRun` directly.
- Use `profile="perception"` when the scenario needs auxiliary services from
  that `antioch.yaml` profile. Use `restart="profile"` or a list of service
  names when an interactive run needs clean service state. Queued runs always
  start a fresh stack and ignore this authoring restart preference.

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
you can branch on it. A run that fails any check finishes `FAILED`; a run that
declares none, or passes them all, finishes `PASSED`. So the outcome is the
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
  criterion replaces its verdict in place, so a per-step check in a control
  loop costs one row and keeps the order you declared.
- **`assert` and `run.fail(reason)` are for "cannot continue"** — the rig did
  not build, the policy file is missing. Both stop the body immediately and
  mark the run `FAILED`.
- **`run.skip(reason)` is for an unmet precondition**, not for a failure.
- **`run.set_outcome(...)` overrides the checks**, for a scenario whose
  judgement is more subtle than a conjunction. An exception overrides both and
  reports `ERRORED` — a defect in the run, not a verdict on the task.

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

Prefer many small cases over one scenario that loops internally: fifty cases
give you fifty records, fifty verdicts, and fifty recordings that a suite can
fan out across machines. One scenario looping fifty times gives you one row.

## Record the evidence

```text
ScenarioRun.add_result(name: str, value: Any) -> None
ScenarioRun.add_results(results: dict[str, Any]) -> None
ScenarioRun.add_artifact(path: str | Path, *, name: str | None = None, content_type: str | None = None) -> None
```

- Results must be valid JSON and fit in the saved scenario results. Put the summary
  there, such as thresholds, counts, and aggregate error, and save a
  per-episode table as an artifact.
- Artifacts upload directly from the machine to object storage. Any media type
  is fine. The name
  `telemetry` is reserved for the session's own `.rrd`.
- Record the thresholds a check used, not just its verdict. A reader six weeks
  later needs to know what "passed" meant.

## Default telemetry

Every managed scenario, and every native `ScenarioSession`, records a
dashboard without a line of telemetry code:

- **A read-back of Kit's active viewport.** Once physics is stepping, the
  platform captures the viewport and logs one JPEG to `/antioch/viewport` on
  the `sim_time` timeline about twice a simulated second, 640 px wide, capped
  at 600 frames per run. What it shows is whatever Kit's viewport camera is
  pointed at, which on a bench-scale workcell may not frame the task. For a
  controlled view, log the scenario's own camera through `Logger.image` at
  `/viewport` or another authored path.
- **A viewer layout derived from what was logged** — a 2D view per entity
  carrying an image or video, `/antioch/viewport` first, one time-series view per
  scalar path, and a 3D view when any spatial entity exists, all under a
  visible container.
- **The viewer's own chrome collapsed out of the way**: the blueprint tree and
  selection inspector open hidden and the time panel collapsed, so the window
  belongs to the telemetry. Name a panel in your own blueprint to keep it.

Capture rides the physics-step callback, never changes the run's outcome, and
reports what it got at the end (`viewport telemetry captured N frames over
X.Xs of simulation, N.N per second`). Turn it off only when frames are
overhead a throughput-critical suite cannot afford:
`@antioch.scenario(capture=False)`, `ScenarioSession(..., capture=False)`, or
`ANTIOCH_TELEMETRY_CAPTURE=0`.

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

```python
logger.scalar("metrics/tilt_deg", tilt_deg)
logger.image("camera/bench", rgb)  # camera frames go here
logger.value("metrics", {"error": error, "reward": reward})
```

**Log pictures through `Logger.image`, never `logger.value(..., rr.Image(...))`.**
`image` downsamples and JPEG-compresses on the way in, the same normalization
the platform's own capture uses; a few hundred raw frames is the difference
between an RRD somebody opens and one they give up downloading.

Sample cameras at a rate a person would watch — a handful of frames per
simulated second — and prefer more frames at a smaller size over a few
enormous ones. A 960-wide JPEG of a lit scene is about 12 KB.

Antioch stamps every write with `wall_time`, and with `sim_time` once Kit is
running. Never call `rr.set_time`, `rr.init`, `rr.connect`, or
`rr.send_blueprint` — the platform owns the recording.

## Choose a layout, or don't

The automatic blueprint is good enough for most runs; author one when the
default ordering hides the thing that matters.

```python
run.set_blueprint(rrb.Blueprint(rrb.Horizontal(rrb.Grid(*views), rrb.Grid(*series))))
```

An author blueprint replaces the automatic one **entirely** — include a view
for every entity you still want visible, `/antioch/viewport` included. Name a
container (`Grid`, `Horizontal`, `Vertical`) as the root: a bare list of views
serializes as a tab container, which shows one pane and hides the rest.
[references/telemetry.md](references/telemetry.md) owns the verified 0.36.0
constructors, `SpatialInformation`, and the live-versus-recorded flow.

## Watch it live

Interactive dispatch streams by default. If another process holds the
machine's one livestream lease, the default mode warns and retries headlessly;
explicit `--stream` requires the lease and fails instead. Use `--no-stream` for
headless work, and remember that queued runs are always headless. There is no
stream size or rate field in simulation code or `antioch.yaml`; **Settings** in
the webapp controls the resolution ceiling and frame rate for the next browser
connection.

## Run it and read it back

```bash
antioch scenario collect                                   # discovery and schema errors, locally
antioch scenario run --scenario vial_place --stream        # one scenario, watched
antioch suite run smoke                                    # a declared suite
antioch scenario show SCENARIO_RUN_ID                      # verdict, checks, results, artifacts
antioch scenario logs SCENARIO_RUN_ID                      # captured output
antioch scenario download SCENARIO_RUN_ID                  # the .rrd and every artifact
```

After changing a scenario's telemetry, capture, checks, or blueprint, prove
what it actually recorded:

1. `antioch scenario show SCENARIO_RUN_ID` — is the outcome the task's
   outcome, and does every criterion you meant to declare appear?
2. `antioch scenario download SCENARIO_RUN_ID`, then `rerun rrd stats <file>`
   and `rerun rrd print <file> | head` — are your entity paths there, and does
   `/antioch/viewport` carry roughly two frames per simulated second?
3. Open it in the viewer when the layout matters. The stored blueprint is what
   a reviewer sees.

**A green suite whose RRD you never opened is not evidence.**

Interactive runs use one machine unless the caller passes `--machines` or
repeats `--machine`. Queued runs use immutable images without development
watch rules; Antioch adds the submitted project tree to the simulation image
before the queue accepts it.

Deeper read-back — filtering run history, per-service logs, artifact keys,
deletion — is the `antioch-platform` skill's scenarios reference; suite
follow-up and cancellation is its suites reference.

## Checklist

- [ ] Every criterion the task defines is a `run.check` with a measured `detail`.
- [ ] `assert` / `run.fail` appear only where the run genuinely cannot continue.
- [ ] Variation is expressed as cases, not as a loop inside one scenario.
- [ ] Thresholds are in `results`; per-episode detail is an artifact.
- [ ] Camera frames go through `Logger.image`.
- [ ] Entity paths are a stable hierarchy a blueprint can select without
      knowing per-run values.
- [ ] The stage has a light, or `/antioch/viewport` will be black.
- [ ] The recording has been opened, not just produced.
