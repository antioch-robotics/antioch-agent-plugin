# Live sessions: `antioch jupyter`, streaming, and direct shells

The Jupyter kernel surface is the agent iteration loop: start one kernel on the
machine, drive it one cell at a time, and stop it when done. A kernel is a
machine resource, not a client's process. It survives the command that made it
so the next cell lands on an Isaac that is already booted, but an unattended
idle kernel is eventually culled.

## The kernel loop

- `antioch jupyter start --json` starts one Isaac kernel on the assigned
  machine and prints its identity (human mode prints the bare kernel id on
  stdout for scripts). `--machine MACHINE` pins it.
- `antioch jupyter cell 'print("ready")'` runs one cell and exits while the
  kernel stays alive. Use `--kernel KERNEL_ID` to choose among several live
  kernels; source can also come from stdin. `--timeout` bounds the cell
  (default 900 seconds) — raise it before a long training or SDG cell rather
  than losing the result.
- `antioch jupyter kernels --json` lists live kernels with state and last
  activity.
- `antioch jupyter show KERNEL_ID` shows one kernel's state, stream ownership,
  and next cell, stop, and stream command.
- `antioch jupyter stop --json --kernel KERNEL_ID` shuts one kernel down and
  releases its GPU memory and livestream lease.

Cells own their boot, exactly like `antioch run` scripts: call `antioch.boot()`
in an early cell, and keep `pxr`, `omni`, `carb`, `isaacsim`, and `isaaclab*`
imports inside function bodies in any imported module.

## Stream a kernel

Declare the machine's livestream before the first cell boots Isaac:

```bash
antioch jupyter start
antioch jupyter stream --kernel KERNEL_ID --json
antioch jupyter cell --kernel KERNEL_ID 'antioch.boot()'
antioch jupyter unstream --kernel KERNEL_ID --json
```

The first command prints `KERNEL_ID`.

The machine has one lease. A streaming kernel and a streaming `antioch run`
cannot fight over the same listener; repeating the same kernel claim is
idempotent, while a different holder is refused. Release it with
`antioch jupyter unstream --kernel KERNEL_ID`. Inside Mission Control,
`jupyter stream` is how a kernel's Isaac GUI reaches the console's livestream
pane (`mission-control.md`).

## Use local JupyterLab with a remote kernel

`antioch jupyter lab --no-open --verbose` runs JupyterLab locally against the
assigned machine's kernel and prints the verified URL. Notebooks, scripts,
and saves are ordinary local
files, and Lab's terminal is a local shell. Only the kernel is remote and it
outlives the Lab process. (Mission Control serves a hosted Lab through its
own gateway instead — do not start one there by hand.) Start the development
session separately when local
edits should reach the running services:

```bash
antioch services up --watch
antioch jupyter lab --no-open --verbose
```

`antioch jupyter lab` prepares `sim` and applies one sync before it opens. Keep
`antioch services up --watch` running only when later local edits should continue
reaching the container. Ctrl-C ends `services up --watch` but leaves the containers and
declared ports running, so use `antioch services down` when the stack should stop. Closing
the notebook or stopping Lab leaves the kernel warm until it is stopped or
culled.

## Direct shells

The `antioch services ssh` command opens a PTY in `sim` by default — when the
project has no sim service, name the service explicitly. It resolves an existing
assignment but does not create a recorded run or retain results. Use
`antioch services exec` for a finite diagnostic command; it has a 120-second
ceiling, so use `antioch run --timeout SECONDS` for longer simulation code.
Use `antioch machine ssh` for a VM-shell diagnostic. See
`machines.md` for direct transfer and service selection.
