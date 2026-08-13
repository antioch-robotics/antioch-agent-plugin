# Authoring scenarios

A scenario is a Python function decorated with `@antioch.scenario` — a 3D integration test Antioch can parameterize, repeat, and evaluate. Decoration never executes the body; the CLI discovers scenarios by reading the files locally. This reference covers the dispatch-facing surface: the decorator, discovery, and the boot/session boundary. Designing the scenario itself — modelling pass/fail with `run.check`, recording results and artifacts, telemetry, and viewer layouts — is the `scenario-design` skill; load it whenever the question is what a run should measure or report rather than how it is declared and dispatched.

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
- `sim=antioch.BootProfile(...)` tunes the boot: `physics_dt` / `render_dt`, `log_level`, `render_quality`, `viewport`, `physics_engine` (`"physx"` by default or experimental Isaac Sim Newton), and `extra_args`. Omit `sim` for the default profile; pass `sim=None` when the scenario needs no runner-owned Kit. Use `--no-stream` for that headless case. Streaming is CLI-only and on by default; the profile has no stream field, and encoder defaults belong to the transport rather than `antioch.yaml`.
- `blueprint=` stores a constant Rerun viewer layout for every run; `run.set_blueprint(...)` inside the body can react to what the run discovered instead. `capture=False` turns off the platform's automatic viewport read-back for a scenario whose throughput cannot afford frames (it defaults to on). The `scenario-design` skill owns both in depth.
- `profile=` activates one Compose profile so an auxiliary service joins this scenario's stack; `restart=` recreates services before an interactive dispatch — `"profile"` or a list of service names. Queued runs always start a fresh stack and ignore this preference.
- The complete keyword surface is `name`, `description`, `tags`, `cases`, `blueprint`, `capture`, `sim`, `profile`, and `restart` — nothing else. `timeout` in particular is a dispatch concern — pass `--timeout` on `scenario run` / `suite run`, never to the decorator.

## Discovery rules

- `antioch.yaml`'s optional `scenario_paths` lists files and directories. When omitted, discovery scans Python files in the `services.sim` build context; when `services.sim` uses an image directly, or the stack has no sim service, discovery starts at the project root (the default `antioch init` project is image-based, so it scans the whole project). Hidden directories, virtualenvs (directories with `pyvenv.cfg`), `node_modules`, common cache/build directories, and paths matched by `.gitignore` or `.dockerignore` are skipped. Directories are searched recursively; use `scenario_paths` for a narrower or different scope.
- **Module scope must import with no simulator installed**: `pxr`, `omni`, `carb`, `isaacsim`, `isaaclab*`, and anything depending on them belong inside the scenario function (the engine skills own this invariant). Top level may use `antioch`, NumPy, stdlib — and must not write files, touch the network, or do sim work. When an Isaac import runs too early, the CLI names the offending module.
- Check what discovery sees with `antioch scenario collect --json`.

## Verdicts, results, and telemetry — one pointer

The body declares the task's criteria with `run.check(criterion, passed, detail=...)`, saves named metrics with `run.add_result(...)`, uploads files with `run.add_artifact(...)`, and streams telemetry through a module-scope `antioch.Logger`. A scenario with no checks reports only "the code did not crash" — the anti-pattern to avoid. The `scenario-design` skill owns all of it: check semantics, outcome derivation (`run.fail`, `run.skip`, `run.set_outcome`), result and artifact rules, `Logger` usage, the automatic viewport capture, and blueprints. Load it before writing or reviewing a scenario body.

## Native scripts and notebooks

`antioch scenario run` starts Isaac before calling a scenario — scenario code never calls `antioch.boot()`. Native scripts (`antioch run`) and notebooks own their lifecycle instead: `antioch.boot()` themselves, and record runs with `antioch.ScenarioSession("name", rrd_path=..., live=...)` as a context manager. `antioch.current_scenario_run()` returns the active handle inside either model and raises `StateError` outside an active run.
