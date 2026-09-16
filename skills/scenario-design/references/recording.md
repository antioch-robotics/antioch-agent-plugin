# Record existing Python work

CLI dispatch chooses and monitors remote execution. Direct calls instead
record work in the caller's process; they do not build services, allocate
compute, or start a simulator.

## Source-backed calls and context blocks

Import a decorated scenario from a real project source file and call it
without the injected run argument:

```python
from src.scenarios import evaluate

results = evaluate(seed=2)
```

The wrapper returns `run.results`. If it declares simulation config, start
the simulator first with the same `SimulationConfig` values. Share a config
between the source module's decorator and notebook startup; `stream=None`
and `stream=False` are not interchangeable after startup.
Use `config=None` for simulator-free work.
Profiles and helper-service restart policy do not apply to direct calls.

Use a context for an undecorated block, including notebook/REPL cells:

```python
import antioch

with antioch.Scenario("inspect", rrd_path="inspect.rrd", capture=False) as run:
    run.add_result("sample_count", 10)
    run.check("complete", True, detail="10 samples recorded")
```

Do not construct `ScenarioRun` directly. The decorator requires a real
source file inside the project; a context does not.
`antioch.current_scenario_run()` returns the active handle or raises
`StateError` outside a run.

## Recording authority

In managed compute, recording uses the session revision and inherits the
process stream grant. It does not create a grant. Source-free managed runs
use the `<kernel>` label and cannot be rerun from history.

Outside managed compute, use a valid full project manifest, including a
service, and existing user/PAT credentials. Admission must succeed before the
body runs. Caller records retain results, telemetry, and artifacts but have
no revision, managed rerun, or managed live stream.

`Scenario(..., control=None)` explicitly selects offline work with no
project/auth read or upload. Missing project or credentials never causes an
automatic offline fallback. Publication errors retain local recovery files;
check the saved record rather than treating local files as uploaded evidence.

## Deadline and cancellation

`recording_timeout_s` defaults to 900 seconds and must be finite, positive,
and at most 86400 seconds. It covers the whole recording, including artifact
publication, without renewal.

Cancellation is cooperative: call `run.raise_if_cancelled()`; finalization
also observes cancellation. Expiry closes the record but does not prove the
caller's Python process stopped.
