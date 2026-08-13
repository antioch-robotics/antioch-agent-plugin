# Authoring scenarios

A scenario is a Python function decorated with `@antioch.scenario` — a 3D integration test Antioch can parameterize, repeat, and evaluate. Decoration never executes the body; the CLI discovers scenarios by reading the files locally.

## The decorator

```python
import antioch


@antioch.scenario(
    tags=["warehouse", "smoke"],
    cases=[antioch.case({"seed": 1, "speed": 1.0}, id="seed-1", tags=["fast"])],
    sim=antioch.BootProfile(physics_dt=1 / 120, render_dt=1 / 60),
)
def aisle_run(run: antioch.ScenarioRun, seed: int = 0, speed: float = antioch.param(1.4, ge=0.1, le=3.0, description="Drive speed")) -> None: ...
```

- Name defaults to the function name; description defaults to the first docstring paragraph. The decorator can override either.
- The first parameter must be annotated `run: antioch.ScenarioRun` (injected). Remaining params need scalar annotations (`bool`/`int`/`float`/`str`/`Literal[...]`) plus defaults — validated locally before any machine is requested. No positional-only params, `*args`, or `**kwargs`.
- `antioch.param(default, ge=..., le=..., description=...)` adds bounds and docs; a plain default is fine too.
- `antioch.case({...}, id=..., tags=[...])` declares one named input set; the ID derives from the overrides (`seed=1`-style) when omitted. Give reusable cases stable IDs. Bare dicts in `cases=` are refused at decoration — every entry is an `antioch.case(...)`.
- Sweeps come from the same helper: `antioch.case(grid={"seed": range(10), "friction": [0.3, 0.8]})` expands a Cartesian grid, and `antioch.case(combinations=[{...}, {...}])` declares correlated rows. A scenario caps at 2,000 expanded cases; narrow the grid past that.
- `sim=antioch.BootProfile(...)` tunes the boot: `physics_dt` / `render_dt`, `log_level`, `render_quality`, `viewport`, `extra_args`. Omit `sim` for the default profile; pass `sim=None` when the scenario needs no runner-owned Kit. Use `--no-stream` for that headless case. Streaming is CLI-only and on by default; the profile has no stream field, and encoder defaults belong to the transport rather than `antioch.yaml`.
- `blueprint=` stores a constant Rerun viewer layout for every run; `run.set_blueprint(...)` inside the body can react to what the run discovered instead. `capture=False` turns off the platform's automatic viewport read-back for a scenario whose throughput cannot afford frames (it defaults to on). The `scenario-design` skill owns both in depth.
- `profile=` activates one Compose profile so an auxiliary service joins this scenario's stack; `restart=` recreates services before an interactive dispatch — `"profile"` or a list of service names. Queued runs always start a fresh stack and ignore this preference.
- The complete keyword surface is `name`, `description`, `tags`, `cases`, `blueprint`, `capture`, `sim`, `profile`, and `restart` — nothing else. `timeout` in particular is a dispatch concern — pass `--timeout` on `scenario run` / `suite run`, never to the decorator.

## Discovery rules

- `antioch.yaml`'s optional `scenario_paths` lists files and directories; when omitted, discovery scans Python files in the `services.sim` build context. Hidden directories, virtualenvs (directories with `pyvenv.cfg`), `node_modules`, common cache/build directories, and paths matched by `.gitignore` or `.dockerignore` are skipped. Directories are searched recursively; use `scenario_paths` for a narrower or different scope.
- **Module scope must import with no simulator installed**: `pxr`, `omni`, `carb`, `isaacsim`, `isaaclab*`, and anything depending on them belong inside the scenario function (the engine skills own this invariant). Top level may use `antioch`, NumPy, stdlib — and must not write files, touch the network, or do sim work. When an Isaac import runs too early, the CLI names the offending module.
- Check what discovery sees with `antioch scenario collect --json`.

## Outcomes, checks, and results

Declare the task's criteria with `run.check(criterion, passed, detail=...)` — one call per criterion the task defines, with the measurement in `detail`:

```python
tilt_deg, speed_mps = measure(scene)
run.check("upright", tilt_deg <= 5.0, detail=f"tilt {tilt_deg:.2f}° <= 5.00°")
run.check("at rest", speed_mps <= 0.01, detail=f"{speed_mps:.4f} m/s")
```

Checks never stop the body, so every criterion still gets measured after an early one fails; re-checking a criterion replaces its verdict in place. Checks land in `results` under the reserved `checks` key, and `antioch scenario show` displays them in a dedicated checks row. A scenario with no checks reports only "the code did not crash" — the anti-pattern to avoid.

- Pass: the function returns normally (return value ignored) and no check failed.
- Fail: any check failed, an assertion fires, or `run.fail("reason")`. The last two stop the body immediately — keep them for "cannot continue" and put measured criteria in `run.check`.
- Skip: `run.skip("reason")` for an unmet precondition, not a failure.
- Error: any other uncaught exception — a defect in the run, not a verdict on the task.
- `run.set_outcome(...)` overrides the derived verdict for a judgement subtler than a conjunction of checks.
- `run.add_result(name, value)` / `run.add_results({...})` — named metrics (str/number/bool/list/dict/None; reusing a name replaces). These are filterable later via `scenario list --result`. **Save important measurements before the final assertion.**
- `run.add_artifact(path, name=..., content_type=...)` — uploads immediately, so it survives a later failure. Combine many small files into one archive.
- Read-only context on the handle: `name`, `scenario_run_id`, `case_id`, `tags`, `params`, `results`, `wall_s`, `sim_s`, `live_uri`.

## Telemetry

```python
logger = antioch.Logger("cube")  # channel prefix: cube/height
logger.scalar("height", z)
logger.value("points", rr.Points3D(...))  # Rerun objects, or dicts of numbers
logger.info("grasp settled")
```

Scalars and values become the run's Rerun `.rrd` telemetry (downloaded by `scenario download`); log lines go to both terminal and Rerun. `scalar()`/`value()` are no-ops without an active run; `info()`/`warning()`/`error()` always print to the terminal and also reach Rerun when a run is active, so library code can log unconditionally.

## Native scripts and notebooks

`antioch scenario run` starts Isaac before calling a scenario — scenario code never calls `antioch.boot()`. Native scripts (`antioch run`) and notebooks own their lifecycle instead: `antioch.boot()` themselves, and record runs with `antioch.ScenarioSession("name", rrd_path=..., live=...)` as a context manager. `antioch.current_scenario_run()` returns the active handle inside either model and raises `StateError` outside an active run.
