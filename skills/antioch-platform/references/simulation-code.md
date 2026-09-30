# Script startup and engine configuration

Keep reusable setup and stepping in functions. A plain script owns startup;
put simulator imports inside functions after it. This bounded Isaac Sim
probe advances one second of physics:

```python
import antioch


def main() -> None:
    """
    Start Isaac Sim and advance one second of physics.
    """

    antioch.start_simulation(antioch.SimulationConfig(physics_dt=1 / 120, render_dt=1 / 60, renderer_quality="balanced"))
    try:
        world = antioch.world()
        world.scene.add_default_ground_plane()
        world.reset()
        for _ in range(120):
            world.step(render=False)
    finally:
        antioch.application().close()


if __name__ == "__main__":
    main()
```

Run it in an existing session with `antioch service exec -- python src/main.py`.
A scenario's runner owns startup; declare configuration on the decorator
instead. A Jupyter kernel starts once and retains its state between cells.
For Lab, use its environment or `SimulationContext` loop, not a second World.

## SimulationConfig

| Field | Contract |
|---|---|
| `log_level` | `fatal`, `error`, `warning`, `info`, or `verbose`; omitted uses the engine default |
| `physics_dt`, `render_dt` | Classic World timing; render period must be an integer multiple of physics period |
| `physics_engine` | `physx` or Isaac's experimental `newton` integration |
| `extensions` | Installed extension IDs enabled before stage construction |
| `renderer_quality` | `performance`, `balanced`, or `quality`; global rendering preset that can change sensor pixels but not sensor resolution |
| `extra_args` | Native Kit arguments appended after Antioch defaults |
| `stream` | Allows streaming when the command passes `--stream`; `False` keeps the process headless even with that flag. Without the flag, streaming is off. |
| `timeout_s` | Scenario execution timeout; an explicit CLI `--timeout` overrides it |
| `viewport_updates` | `None` renders the default viewport while streaming and pauses it otherwise; `True` also renders without a stream, and `False` pauses it even with a stream |
| `keyboard_navigation` | `True` (default) flies a streamed viewport's camera from the keyboard: arrows move, `Shift` + arrows look, `Ctrl` + `Up`/`Down` or `Shift` + `E`/`Q` rise and descend, `-`/`=` scale speed, held `X`/`Z` double or halve it. `False` leaves those keys to authored input such as Isaac Lab's `Se2Keyboard`; headless processes never subscribe |

Identical startup is a no-op; conflicting configuration requires a fresh
process or kernel. A closed Kit cannot restart in the same process. Install
additional packages/extensions in the service image before launching it.
An import path need not equal the owning extension ID.

Scene properties, sensors, controllers, and backend-specific behavior belong
to native APIs. Lab timing uses `SimulationCfg`, render interval, and
decimation. `service exec` ends when its command exits unless its own
`--timeout` is set. Use [scenario design](../../scenario-design/SKILL.md)
when work needs recorded inputs, checks, and evidence.

## Access native state

After startup, `antioch.application()` returns Isaac's `SimulationApp`, `antioch.stage()` returns the active OpenUSD stage, and `antioch.world()` returns Isaac Sim's classic `World`. They expose native objects; they do not copy or translate the scene. `antioch.is_running()` checks startup state, and `antioch.engine()` identifies the runtime. Lab owns its own context and environment lifecycle, so use [Lab's stepping contract](../../isaac-lab-3/SKILL.md#startup-and-stepping) there.

Native scripts own shutdown. A dispatched scenario's runner owns shutdown, while a notebook keeps Kit alive for subsequent cells. Do not close Kit after each cell or attempt to start it again in the same closed process.

## Choose rendering and observation

`renderer_quality` sets global RTX settings and the maximum stream image budget: `performance` favors throughput, `balanced` is the default, and `quality` favors the picture. Its RTX settings, including DLSS mode, can change sensor render products that use those settings, so keep the preset fixed when comparing sensor images. It does not resize sensor cameras, set physics rate, or guarantee a video frame rate. Configure a sensor with its native API and choose physics/render timing for the experiment independently.

`extensions` enables installed Kit extension IDs before scene construction. `extra_args` adds native Kit flags after Antioch defaults; verify each flag against the pinned runtime with [research](../../antioch-research/SKILL.md). Neither setting installs missing dependencies.

Use [navigation and capture](../../agentic-simulation/references/viewport.md) to position an inspection camera and see images, [camera guidance](../../isaac-sim-6/references/isaac-camera.md) for sensor data, and [telemetry](../../scenario-design/references/telemetry.md) to save evidence. [Antioch platform](../SKILL.md#capability-guides) links setup and execution.

The stream budgets are 1920×1088 for `performance`, 2560×1440 for `balanced`, and 3840×2176 for `quality`. Headless work uses a 1280×720 main viewport unless native arguments change it. Antioch has no FPS setting; video delivery rate is separate from physics and rendering steps. Native settings applied later remain under the program's control.

Set `viewport_updates=True` for headless code that depends on a continuously rendering default viewport, such as a Replicator orchestrator. Independent sensor render products do not depend on that default viewport policy. Rendering still needs Kit updates: a Lab step without a Kit visualizer may require explicitly driving Kit. [Rendering guidance](../../isaac-sim-6/references/isaac-sim-rendering.md) explains native renderer choices, and [Jupyter](../../agentic-simulation/references/jupyter.md) explains cooperative updates between cells.

A clean script exit returns 0, an uncaught exception returns 1, and `KeyboardInterrupt` returns 130. Use `sys.exit(n)` for an explicit status after startup; a bare top-level `raise SystemExit(n)` can lose the intended code through Isaac's headless shutdown. Chain any replacement `sys.excepthook` if it must preserve Antioch's error handling.
