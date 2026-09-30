---
name: scenario-design
version: "1.4.15"
description: Use when authoring Antioch scenarios, typed cases, measured checks, results, artifacts, or telemetry. Covers evaluation and recording; use antioch-platform for dispatch/history and the Isaac skills for native simulator code.
---

# Scenario design

A scenario evaluates behavior with typed inputs, measured checks, and saved
evidence. Dispatch captures its output and pins images and source; use
[research](../antioch-research/SKILL.md) to ground native measurement APIs.

## Author the unit

```python
import antioch

logger = antioch.Logger("task")


@antioch.scenario(tags=["smoke"], cases=[antioch.case({"seed": 2}, id="seed-2")])
def evaluate(run: antioch.ScenarioRun, seed: int = 1) -> None:
    """Evaluate the requested behavior."""
    # Build, step, and measure the task here.
```

The first parameter is `run: antioch.ScenarioRun`. Remaining parameters need
scalar annotations (`bool`, `int`, `float`, `str`, or `Literal`) and
defaults; `antioch.param(default, ge=..., le=..., description=...)` adds bounds and documentation. Use `Literal` for a closed set of choices. The parameter names, types, defaults, bounds, and descriptions appear during collection so callers can inspect the contract before running it.

The runner starts Kit before the body. Declare startup with
`config=antioch.SimulationConfig(...)`, not a startup call inside the
scenario; `config=None` skips runner-owned Kit but still uses remote compute
under CLI dispatch. Keep simulator imports inside functions; discovery runs
locally without Isaac.

Keep module imports free of simulation, network, or file-writing effects.
`antioch scenario collect --json` validates definitions, not the body;
`scenario_paths` in the manifest limits discovery; without it, discovery scans the project. `antioch.scenarios()` reads definitions already registered in the current process, whereas [client collection](../antioch-platform/references/python-client.md#collect-inspect-and-submit) discovers and expands project source.

## Inputs and execution policy

`antioch.case` accepts parameter overrides, a Cartesian `grid`, or
correlated `combinations`; use helper objects, not bare dictionaries, in
`cases=`. IDs can format resolved parameters; omitted IDs are derived.
Expansion is capped at 2,000 cases per scenario. Use cases for independent
outcomes and comparisons, or an internal loop for one evaluation/dataset
lifecycle; [suite selectors](../antioch-platform/references/suites.md) group
cases in YAML.

```python
cases = [
    antioch.case({"speed": 0.2}, id="slow", tags=["smoke"]),
    antioch.case(grid={"speed": [0.2, 0.4], "seed": [1, 2]}, id="speed-{speed}-seed-{seed}"),
    antioch.case(combinations=[{"speed": 0.2, "load_kg": 20.0}, {"speed": 0.4, "load_kg": 5.0}]),
]
```

A grid crosses independent values; combinations preserve intentional pairs. Shared overrides can accompany either expansion, but a parameter cannot be set both in the shared overrides and the expansion. Case tags combine with scenario tags for selection. Keep IDs stable enough to compare the same case across revisions.

Decorator keywords: `name`, `description`, `tags`, `cases`, `config`,
`blueprint`, `capture`, `service`, `profile`, and `restart_services`.

`service` selects an eligible active container; it does not activate a profile. Omit it only when the active graph has one eligible service. `profile` includes helper services in the dispatched revision; `restart_services` restarts their processes and waits for fresh health before each managed run. Direct calls apply neither policy. For development with those helpers, or to target a session explicitly, start the required graph with `antioch session new --profile perception`. A submitted bundle reaches every mapped service, but a service outside `restart_services` keeps its existing process. If its files differ from those that process started with, the run reports earlier service code. Restart that helper between runs or add it to this policy. See [manifest services](../antioch-platform/references/manifest.md#services-and-images) and [restart behavior](../antioch-platform/references/sessions.md#run-and-sync-source).

## Measured verdicts

```python
run.add_result("max_error_m", max_error_m)
run.add_result("error_limit_m", error_limit_m)
run.check("position", max_error_m <= error_limit_m, detail=f"{max_error_m:.4f} m")
run.add_artifact("outputs/measurements.json", description="Measured task trajectory")
```

Use one named check per task criterion, with measured detail. Checks return
the supplied boolean and do not stop execution; reusing a criterion replaces
its verdict, so accumulate failures or worst-case values for conditions that
must hold throughout motion. Geometric clearance is not measured contact, and
a commanded grasp is not a measured lift; label proxies as proxies.

On normal completion, any failed check gives `FAILED`; all passing checks or
no checks give `PASSED`, so no checks proves only that execution completed.
`run.fail` and assertion failures stop the body with `FAILED`; `run.skip`
reports an unmet precondition; `run.set_outcome` overrides checks, but
unexpected exceptions still produce `ERRORED`.

Keep thresholds and summaries in JSON results and do not write the reserved `checks` key. `run.add_results({...})` stores several results together; values must be finite, JSON-compatible values, so convert arrays and tensors before storing a small summary. Keep large arrays, tables, images, and files in artifacts instead. Artifact names are unique within a run; use `name=` when files share a basename. `output` and `telemetry` are reserved artifact names. An artifact uploads when `add_artifact` is called; keep many small files in one archive to avoid an upload for each.

`run.params`, `case_id`, `tags`, `scenario_run_id`, `suite_run_id`, `suite`, and `suite_position` describe the current evaluation. `wall_s` and `sim_s` distinguish elapsed wall time from simulation time; `sim_s` can be absent without a simulator. Long loops should call `run.raise_if_cancelled()` so a requested cancellation can stop useful work promptly. [In-session recording](references/recording.md#cancellation-and-process-exit) explains why a direct call cannot rely on the platform killing its parent process.

## Telemetry and read-back

A module-scope `antioch.Logger` resolves the active run on each call. Use
`scalar` for metrics, `image` for raw RGB/RGBA review frames, and `value`
for Rerun archetypes; image calls take a path and then pixels:
`logger.image("camera/front", rgb)`. Read [telemetry](references/telemetry.md)
for image encoding, timelines, geometry, layouts, and diagnosing RRDs.

Automatic viewport capture is a bounded diagnostic sampler, not a task check:
it neither aims the camera nor proves physical correctness. `capture=False`
disables it.

After execution, inspect the checks, results, and relevant artifacts through
[run history](../antioch-platform/references/scenarios.md). Smooth video
alone does not establish physical correctness.

For imported scenario calls or notebook recording inside a session, read
[in-session recording](references/recording.md). Native simulator APIs belong
to [Isaac Sim](../isaac-sim-6/SKILL.md), [Isaac Lab](../isaac-lab-3/SKILL.md),
and [research](../antioch-research/SKILL.md).

## Choose the next reference

| Task | Load when needed |
|---|---|
| Choose typed inputs, cases, or a pass/fail criterion | This skill's authoring, input, and measured-verdict sections |
| Save camera images, metrics, geometry, and a useful viewer layout | [Telemetry and viewer layouts](references/telemetry.md) |
| Record code in a script or notebook; handle offline work or cancellation | [In-session recording](references/recording.md) |
| Run, compare, download, cancel, or repeat saved work | [Scenario history](../antioch-platform/references/scenarios.md) and [suites](../antioch-platform/references/suites.md) |
| Iterate on native code before making a recorded evaluation | [Agentic simulation](../agentic-simulation/SKILL.md) |
