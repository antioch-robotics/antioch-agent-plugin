---
name: scenario-design
version: "1.4.4"
description: >-
  Guides Antioch evaluation design: scenarios, typed parameters and cases, measured checks, results, artifacts, caller recording, telemetry, and Rerun layouts. Use when authoring or reviewing scenarios and suites, defining pass/fail criteria, recording experiments, or diagnosing saved evidence and viewer output. Use antioch-platform for dispatch/history and the Isaac skills for simulator code.
---

# Scenario design

A scenario is a Python evaluation with inputs, a task, measured verdicts,
and evidence. Managed execution pins its service images and captures process
output. A useful result explains what happened and why it passed or failed.
See [research](../antioch-research/SKILL.md) to ground the physical model and
measurement APIs, and [agentic simulation](../agentic-simulation/SKILL.md) to
iterate on experiments, including fitting parameters to real measurements.

## Author the unit

```python
import antioch

logger = antioch.Logger("task")


@antioch.scenario(tags=["smoke"], cases=[antioch.case({"seed": 2}, id="seed-2")])
def evaluate(run: antioch.ScenarioRun, seed: int = 1) -> None:
    """Evaluate the requested behavior."""
    # Build, step, and measure the task here.
```

The first parameter is `run: antioch.ScenarioRun`. Remaining parameters
need scalar annotations (`bool`, `int`, `float`, `str`, or
`Literal`) and defaults. Use `antioch.param(default, ge=..., le=...,
description=...)` for bounds and documentation.

The runner starts Kit before the body. Declare startup with
`config=antioch.SimulationConfig(...)`, not a startup call inside the
scenario. `config=None` skips runner-owned Kit but still uses remote compute
under CLI dispatch. Keep simulator imports inside functions; discovery runs
locally without Isaac.

`scenario_paths` in the manifest selects files/directories for discovery;
omitting it scans the project. Keep module imports free of simulation, network,
or file-writing effects. Preview with `antioch scenario collect --json`.
Collection validates definitions, not the body. Verify native setup and task
logic with a bounded runtime probe before expanding to many cases.

## Inputs and execution policy

`antioch.case` accepts parameter overrides, a Cartesian `grid`, or
correlated `combinations`. Use helper objects, not bare dictionaries, in
`cases=`. IDs can format resolved parameters; omitted IDs are derived.
Expansion is capped at 2,000 cases per scenario.

Use cases for independent outcomes and comparisons, or an internal loop for
one evaluation/dataset lifecycle. [Suite selectors](../antioch-platform/references/suites.md)
group cases in YAML.

Decorator keywords: `name`, `description`, `tags`, `cases`, `config`,
`blueprint`, `capture`, `profile`, `restart_services`, `recording_timeout_s`.

Inspect the installed signature for defaults and types.
For background dispatch, `profile` includes its helper services in the revision.
For interactive work, select profiles when starting the session with
`antioch session new --profile perception`; dispatch reuses that session's revision.
`restart_services` restarts those processes with fresh health before each
background child. Interactive execution and direct calls do not apply it.

## Measured verdicts

```python
run.add_result("max_error_m", max_error_m)
run.add_result("error_limit_m", error_limit_m)
run.check("position", max_error_m <= error_limit_m, detail=f"{max_error_m:.4f} m")
run.add_artifact("outputs/measurements.json", description="Measured task trajectory")
```

Use one named check per task criterion, with measured detail. Checks return
the supplied boolean and do not stop execution. Reusing a criterion replaces
its verdict: accumulate failures or worst-case values for conditions that
must hold throughout motion.

Match evidence to the claim: geometric clearance is not measured contact,
and a commanded grasp is not a measured lift. Label proxies as proxies.

On normal completion, any failed check gives `FAILED`; all passing checks
or no checks give `PASSED`. No checks proves only that execution completed.
Derive task summaries from all required checks; a successful planner or
substep does not cancel a failed task criterion.
`run.fail` and assertion failures stop the body with `FAILED`;
`run.skip` reports an unmet precondition. `run.set_outcome` overrides
checks, but unexpected exceptions still produce `ERRORED`.

Keep thresholds and summaries in JSON results. Do not write the reserved
`checks` key. Put detailed tables, images, and other files in artifacts;
`output` and `telemetry` are reserved platform artifact names. Preserve
failed measurements and images.

## Telemetry and read-back

A reusable module-scope `antioch.Logger` resolves the active run on each
call. Use `scalar` for metrics, `image` for raw RGB/RGBA review frames,
and `value` for Rerun archetypes. Image calls take a path and then pixels:
`logger.image("camera/front", rgb)`. Read [telemetry](references/telemetry.md)
when choosing image encoding, timelines, geometry, layouts, or diagnosing RRDs.

Automatic viewport capture is a bounded diagnostic sampler, not a task check.
It neither aims the camera nor proves physical correctness. Disable it with
`capture=False` when dedicated evidence makes it unnecessary.

After execution, read terminal checks/results and the artifacts needed for the
claim with `antioch scenario list`, `show`, and `download`;
see [run inspection](../antioch-platform/references/scenarios.md).
Inspect decoded images and timestamps when relevant. Run counts or smooth
video alone do not establish repeatability or correct physical behavior.

For imported scenario calls or notebook/local recording, read
[caller recording](references/recording.md). Native simulator APIs belong to
[Isaac Sim](../isaac-sim-6/SKILL.md), [Isaac Lab](../isaac-lab-3/SKILL.md),
and [research](../antioch-research/SKILL.md).
