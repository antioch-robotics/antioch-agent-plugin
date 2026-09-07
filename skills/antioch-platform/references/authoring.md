# Authoring scenarios

A scenario is a Python function decorated with `@antioch.scenario` — a 3D integration test Antioch can parameterize, repeat, and evaluate. Decoration never executes the body; the CLI discovers scenarios by reading the files locally. This reference covers the dispatch-facing surface: the decorator, discovery, and the simulation/session boundary. Designing the scenario itself — modelling pass/fail with `run.check`, recording results and artifacts, telemetry, and viewer layouts — is the `scenario-design` skill; load it whenever the question is what a run should measure or report rather than how it is declared and dispatched.

## The decorator

```python
import antioch


@antioch.scenario(
    tags=["warehouse", "smoke"],
    cases=[antioch.case({"seed": 1, "speed": 1.0}, id="seed-1", tags=["fast"])],
    config=antioch.SimulationConfig(physics_dt=1 / 120, render_dt=1 / 60),
    restart_services=("autonomy",),
)
def aisle_run(run: antioch.ScenarioRun, seed: int = 0, speed: float = antioch.param(1.4, ge=0.1, le=3.0, description="Drive speed")) -> None: ...
```

- Name defaults to the function name; description defaults to the first docstring paragraph. The decorator can override either.
- The first parameter must be annotated `run: antioch.ScenarioRun` (injected). Remaining params need scalar annotations (`bool`/`int`/`float`/`str`/`Literal[...]`) plus defaults — validated locally before anything is dispatched. No positional-only params, `*args`, or `**kwargs`. The imported name is a **programmatic call** wrapper: it omits the run and returns `run.results`. Inside a managed session command, the call creates a saved scenario run. Outside managed compute, it remains local and temporary. CLI dispatch still calls the original body.
- `antioch.param(default, ge=..., le=..., description=...)` adds bounds and docs; a plain default is fine too.
- `antioch.case({...}, id=..., tags=[...])` declares one named input set; the ID derives from the overrides (`seed=1`-style) when omitted. Give reusable cases stable IDs. Bare dicts in `cases=` are refused at decoration — every entry is an `antioch.case(...)`.
- Sweeps come from the same helper: `antioch.case(grid={"seed": range(10), "friction": [0.3, 0.8]})` expands a Cartesian grid, and `antioch.case(combinations=[{...}, {...}])` declares correlated rows. A scenario caps at 2,000 expanded cases; narrow the grid past that.
- `config=antioch.SimulationConfig(...)` configures startup: `physics_dt` / `render_dt`, `log_level`, `render_quality`, `viewport`, `physics_engine` (`"physx"` by default or experimental Isaac Sim Newton), `extensions`, `extra_args`, and optional `stream` and `timeout_s` defaults. The engine keeps upstream Kit defaults; opt into ROS 2 with `extensions=("isaacsim.ros2.bridge",)` and physics sensors with `extensions=("isaacsim.sensors.experimental.physics",)`. Custom extension ids use the same tuple. Omit `config` for the default config; pass `config=None` when the scenario needs no runner-owned Kit. Attached `antioch scenario run` and `antioch suite run` inherit these values when their CLI flags are omitted; explicit `--stream`/`--no-stream` and `--timeout` flags override them. A multi-scenario selection with mixed or partly missing authored defaults must pass one explicit flag instead of silently discarding a value. A native script's `start_simulation()` honors its authored `stream` default, while the outer `services exec` command deadline remains its explicit `--timeout` (or 900-second) process bound because they accept arbitrary commands. Detached work is headless. Encoder defaults belong to the transport rather than `antioch.yaml`.
- `blueprint=` stores a constant Rerun viewer layout for every run; `run.set_blueprint(...)` inside the body can react to what the run discovered instead. `capture=False` turns off the platform's automatic viewport read-back for a scenario whose throughput cannot afford frames (it defaults to on). The `scenario-design` skill owns both in depth.
- `profile=` names one `antioch.yaml` profile; the revision a scenario submission freezes includes that profile's auxiliary services.
- `restart_services=(...)` declares background policy. Before a background child runs, Antioch restarts those named services from the pinned revision and waits for health. Interactive execution ignores this declaration because it is the user's live environment; restart there with `antioch services restart`.
- The complete keyword surface is `name`, `description`, `tags`, `cases`, `blueprint`, `capture`, `config`, `profile`, and `restart_services`. `profile` names a Compose profile; `config` is the typed simulation launch config. Check `antioch scenario run --help` before setting CLI defaults.

## Discovery rules

- `antioch.yaml`'s optional `scenario_paths` lists files and directories where `@antioch.scenario` entry points live. Directories are searched recursively for those decorated files only — helper modules beside them are not imported unless a scenario imports them. Files inside a regular or namespace package are loaded as package members, package initializers run as they would for a normal import, and `from .helper import ...` resolves. When `scenario_paths` is omitted, discovery scans from the project root. Hidden directories, virtualenvs (directories with `pyvenv.cfg`), `node_modules`, common cache directories, and paths matched by `.gitignore` are skipped.
- **Module scope must import with no simulator installed**: `pxr`, `omni`, `carb`, `isaacsim`, `isaaclab*`, and anything depending on them belong inside the scenario function (the engine skills own this invariant). Top level may use `antioch`, NumPy, stdlib — and must not write files, touch the network, or do sim work. When an Isaac import runs too early, the CLI names the offending module. A file that genuinely cannot be imported names the file, the failing import, and what to change, and does not abort discovery of the rest.
- Check what discovery sees with `antioch scenario collect --json`.

## Verdicts, results, and telemetry — one pointer

The body declares the task's criteria with `run.check(criterion, passed, detail=...)`, saves named metrics with `run.add_result(...)`, uploads files with `run.add_artifact(...)`, and streams telemetry through a module-scope `antioch.Logger`. A scenario with no checks reports only "the code did not crash" — the anti-pattern to avoid. The `scenario-design` skill owns all of it: check semantics, outcome derivation (`run.fail`, `run.skip`, `run.set_outcome`), result and artifact rules, `Logger` usage, the automatic viewport capture, and blueprints. Load it before writing or reviewing a scenario body.

## Native scripts and notebooks

A CLI-dispatched run (`antioch scenario run`) starts Isaac before calling a scenario — scenario code never calls `antioch.start_simulation()`. Scripts run with `antioch services exec` and notebooks own their lifecycle instead: they call `antioch.start_simulation()`. After startup, import a source-backed decorated scenario and make a **programmatic call**; the caller owns Kit and the wrapper verifies a live matching config:

```python
from src.scenarios import aisle_run

results = aisle_run(speed=1.2)
```

Inside a managed session command, context entry creates and updates the saved run. Outside managed compute, text is immediate, telemetry is temporary, and `run.add_artifact(...)` is unavailable. Use CLI dispatch when Antioch must select and monitor the execution. Keep `antioch.Scenario("name", rrd_path=..., livestream=...)` as a context manager for an undecorated block. Defining `@antioch.scenario` in a notebook cell is still refused because the decorator needs a real source file. `antioch.current_scenario_run()` returns the active handle inside either model and raises `StateError` outside an active run.
