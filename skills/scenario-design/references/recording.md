# Record Python work inside a session

Direct calls record work in the calling process, which must run inside a
session: a command started with `antioch service exec`, or a shell or
notebook on the session. They do not build services, allocate compute, or
start a simulator.

## Source-backed calls and context blocks

Import a decorated scenario from a real project source file and call it
without the injected run argument:

```python
from src.scenarios import evaluate

results = evaluate(seed=2)
```

The wrapper returns `run.results`. If it declares simulation config, start
the simulator first with the same `SimulationConfig` values; the running
stream is accepted whatever `stream` says. Use `config=None` for
simulator-free work. Profiles and helper-service restart policy do not
apply to direct calls.

Request a stream on the cell that starts Isaac with `antioch jupyter cell --stream`. Later cells reuse the running engine and cannot change its stream mode. Call the scenario in that kernel:

```bash
antioch jupyter cell 'from src.scenarios import evaluate; results = evaluate(seed=2); print(results)'
```

Direct calls also retain the kernel's existing scene and Python state. The
wrapper does not clear them between runs. Make repeated calls reset the
objects they own, or use a clean kernel for a function that always constructs
a new scene. Do not clear an unrelated scene merely to make a call succeed.
For an Isaac Sim function that owns the whole scene, call `world.stop()` before
`world.clear()`, then rebuild the objects and call `world.reset()`. Clearing a
playing world can leave invalid physics views on the next call.

Use a context for an undecorated block, including notebook/REPL cells:

```python
import antioch

with antioch.Scenario("inspect", rrd_path="inspect.rrd", capture=False) as run:
    run.add_result("sample_count", 10)
    run.check("complete", True, detail="10 samples recorded")
```

Do not construct `ScenarioRun` directly. The decorator requires a real source
file inside the project; a context does not.
`antioch.current_scenario_run()` returns the active handle or raises
`StateError` outside a run.

## Recording authority

Recording uses the session's credentials and inherits the process stream
grant; it does not create one. Admission must succeed before the body runs.
The run is bound to the calling process: it runs beside that process's other
work instead of queueing behind the session's runs, and it has no managed
rerun. Source-free runs use the `<kernel>` label.

Outside a session, a direct call stops with one line naming
`antioch scenario run NAME` and `antioch service exec`.
`Scenario(..., control=None)` explicitly selects offline work with no
project/auth read or upload; a missing session never causes an automatic
offline fallback. Publication errors retain local recovery files; check the
saved record rather than treating local files as uploaded evidence.

## Cancellation and process exit

Cancellation is cooperative: call `run.raise_if_cancelled()`; finalization
also observes cancellation. Antioch never signals the calling process. It
closes a cancelled run shortly after if the process does not report, and it
closes the run with its saved results when the process exits or the session
ends first.

Offline recording can retain a local RRD and results, but `run.add_artifact` requires a managed record and refuses offline uploads. Use [telemetry](telemetry.md) for the saved file's contents, [Jupyter](../../agentic-simulation/references/jupyter.md) for kernel lifecycle, and [scenario design](../SKILL.md#choose-the-next-reference) for typed definitions and checks. To turn the experiment into repeatable fresh-process work, use [managed dispatch](../../antioch-platform/references/scenarios.md).
