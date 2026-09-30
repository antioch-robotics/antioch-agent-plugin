# Iterate in a Jupyter kernel

The `antioch-jupyter` MCP runs stateful Python cells in a session's simulator
service.

## The loop

Have a running session for the project (`antioch session new`); the MCP never starts one. `jupyter_connect()` defaults to your only running session in the project that you started yourself. Pass `session=SESSION_ID` when you have several or want a session Antioch started for a run. This tool has no `run` argument: read the session ID with `antioch scenario show SCENARIO_RUN_ID --json` and pass it explicitly. Then use the returned `kernel.id` as `kernel_id` in the other tools.

Boot the simulator once per kernel. To call a decorated scenario, use its
declared `SimulationConfig`; a conflicting startup config is refused.
Use `log_level="warning"` when normal startup messages make the cell difficult to inspect:

```python
import antioch

antioch.start_simulation(antioch.SimulationConfig(log_level="warning"))
```

The MCP does not request video. To watch Isaac in Mission Control, use `antioch jupyter cell --stream` for the cell that first starts it, including in an existing kernel. If Isaac has already started headless, restart the kernel before changing its stream mode. [Simulation configuration](../../antioch-platform/references/simulation-code.md#simulationconfig) explains the startup settings.

Keep the code you are developing in project modules and drive it from short
cells; the kernel runs in the project directory:

```python
import importlib

from src import trial

importlib.reload(trial)
print(trial.run(drop_height=2.0))
```

Edit the module locally, reload, call again. Whether you or Antioch started the session, `jupyter_execute` syncs your local files before each cell. On a session executing a run, this can change files the run or its helper services read later. Avoid concurrent edits when preserving that run's inputs matters; see [source sync](../../antioch-platform/references/sessions.md#run-and-sync-source). To see the scene,
`from antioch.lib import viewport` and `display(await viewport.observe_async())`
show the live frame inline; see [navigation and capture](viewport.md).

## Control one kernel

| Tool | Use |
|---|---|
| `jupyter_connect(session=..., service=..., kernel_id=...)` | Select a session/service and reuse or create a kernel. Omit optional selectors only when the choice is unambiguous. |
| `jupyter_execute(kernel_id=..., code=..., timeout_s=...)` | Run a cell and return text, tracebacks, and displayed PNG/JPEG images. Variables and top-level `await` work between cells. |
| `jupyter_kernel(kernel_id=..., action="interrupt")` | Ask the current cell to stop; inspect its eventual result before repeating work. |
| `jupyter_kernel(kernel_id=..., action="restart")` | Start a fresh Python process and lose variables; needed for a new Kit startup configuration. |
| `jupyter_kernel(kernel_id=..., action="stop")` | Shut down only that kernel, leaving the session running. |

Use the exact returned kernel ID. Restart can return a new kernel record, so use its ID for later calls. A timeout requests interruption but may not prove the cell stopped. If the adapter reports an unconfirmed stop, restart the kernel before another execution. Do not create another kernel just to bypass uncertain side effects.

`jupyter_execute` defaults to 60 seconds and allows at most 900 seconds; choose an explicit startup budget because boot or a first rendered frame can exceed the default. The CLI cell command has its own 900-second default. A busy kernel refuses another execution until the cell ends or an explicit interruption settles.

Tool output is bounded. Display a few relevant frames, summarize arrays, and save large results to files or [run artifacts](../../scenario-design/SKILL.md#measured-verdicts). To record an imported scenario in the same kernel, follow [in-session recording](../../scenario-design/references/recording.md); direct calls retain existing scene state.

## Keep the scene, reset the state

Build the scene once and reset between trials. In Isaac Sim, `world.reset()`
returns registered objects to their initial poses but steps physics to do so:
a dropped ball came back with -0.33 m/s, two 60 Hz steps of gravity. Set
velocities, joint targets, and anything else a trial changed when the starting
state matters. Keep scene construction and one trial in separate functions.

To rebuild instead, stop first: `world.stop()`, `world.clear()`, build, then
`world.reset()`. Adding a rigid body while the stage is still simulating fails
with `Failed to get rigid body velocities from backend`. RTX sensors do not
survive a clear: a rebuilt lidar returned zero scans, so restart the kernel
to rebuild a scene with sensors.

In Isaac Lab, build the environment once and call `env.reset()` between
trials. A second `gym.make` after `env.close()` failed in the same kernel with
`Unable to retrieve replicator graph`, so restart the kernel to change the
environment's configuration.

## Long cells and failures

- The first rendered frame has taken up to five minutes; allow for it, for
  example with `timeout_s=600`. A timeout during startup or rendering can
  leave the kernel needing a restart.
- After a timeout or lost connection the cell may have partly run; check
  state before repeating anything with side effects.
- Kit boots once per Python process; changing its startup settings needs a
  kernel restart.
- The kernel pumps rendering, not physics: step physics from your code and
  `await` inside long asynchronous operations.

A warning repeated every step can bury a cell's output. Silence its channel:

```python
import omni.log

omni.log.get_log().set_channel_level("omni.physx.plugin", omni.log.Level.ERROR, omni.log.SettingBehavior.OVERRIDE)
```

## Finish

Save what you need to project files or run artifacts. A kernel retains its state until it is stopped or its session ends. A running cell keeps the session active. An open client connection through the forwarded Jupyter port also keeps it active, but leaving the MCP adapter open is not a reservation. An idle Jupyter server or kernel alone does not prevent release. Once the session has no work or open client connections, its idle timeout applies. `antioch jupyter stop` stops the server and kernels immediately; `antioch session release` releases the compute.

Return to [agentic simulation](../SKILL.md#choose-an-execution-loop) to choose between this stateful loop, a clean script process, and managed runs. [Session guidance](../../antioch-platform/references/sessions.md#jupyterlab) covers the CLI equivalent; [viewport capture](viewport.md) covers inline inspection.
