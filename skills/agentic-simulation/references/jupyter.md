# Interactive Python

The `antioch-jupyter` MCP executes ordinary Python cells in the project's
remote simulator service. Start its adapter, `antioch-jupyter-mcp`, from the
owning project directory with the intended SDK environment.

## Connect and execute

First use the platform's [session workflow](../../antioch-platform/references/sessions.md)
to obtain the authorized interactive session. The adapter never allocates one.

| Tool | Contract |
|---|---|
| `jupyter_connect(kernel_id=...)` | Connect to the project session; start Lab and a kernel if needed. Omit the ID to select the sole kernel |
| `jupyter_kernels()` | List existing kernels without creating one |
| `jupyter_execute(kernel_id=..., code=..., timeout_s=...)` | Execute one arbitrary cell and return its text, JSON, and images |
| `jupyter_kernel(kernel_id=..., action=...)` | Interrupt, restart, or stop the exact kernel |
| `jupyter_disconnect()` | Close the adapter connection without stopping kernels or compute |

Read the returned project/session identity and retain the exact kernel ID.
When selection is ambiguous, list kernels and reconnect with one returned ID.
The adapter's launch directory fixes its project; a later shell `cd` does
not retarget it. Relaunch after a project, identity, or deployment change.
If the host cannot launch it at the project root, use project-root
`antioch jupyter cell` and copy saved images for inspection.

## Work in the kernel

For a notebook-owned simulator, start once before native imports. If calling
a decorated scenario, use its exact declared configuration instead of the
example below; see [caller recording](../../scenario-design/references/recording.md).

```python
import antioch

antioch.start_simulation(antioch.SimulationConfig(stream=False))
```

`stream=False` keeps the GUI producer free; viewport capture still works.
The kernel pumps rendering, not physics. Code owns physics steps and must
yield during long asynchronous operations. Keep Kit alive between cells;
closing it requires a kernel restart before another boot.

Import saved project modules, call bounded experiments, and explicitly print
state or display images. The adapter adds no simulation code or automatic
capture. See [viewport inspection](viewport.md) for Python helpers.

## Recover and finish

Execution defaults to a 60-second timeout and accepts up to 900 seconds.
Allow longer for startup and first rendering. A deadline leaves completion
unknown: inspect the kernel or interrupt the owned cell before deciding
whether to run more code. Interrupt is not proof that native execution stopped.

Wait for the active adapter call to finish or reach its deadline before
restart or stop; busy calls are refused. Restart discards all Python and simulator state; rebuild from saved
source. Stop affects the selected kernel, not the session. Avoid sharing a
mutable kernel with another writer.

An idle adapter may disconnect after about 60 seconds. Reconnect to the same
kernel; do not replay initialization merely because transport closed.
Save useful source and evidence, disconnect when no call is active, then use
`antioch jupyter lab --stop` and release the owned session when finished.
Disconnect leaves Lab running; session release refuses while it has active work.
