# Isaac Sim 6 Troubleshooting — Large USD Scene Hangs

Adapted from NVIDIA [`isaac-sim-troubleshooting/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-troubleshooting/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Diagnose Isaac Sim startup hangs, stage-load stalls, MDL/shader compilation freezes, physics stepping blocks, Replicator/Hydra issues, Nucleus latency, and GPU OOM crashes.

## Limitations

- Symptom lists are heuristic; root causes can overlap across subsystems.
- Some upstream fixes target local Kit builds or NVIDIA-internal packages and do not apply to the managed service.

## Troubleshooting

| Error / symptom | Cause | Solution |
|---|---|---|
| Extension or import not found | Optional extension not enabled, or startup failure | Enable the owning extension ID via `SimulationConfig.extensions`; check startup logs |
| Black or empty frames | Missing lights or non-RTX render mode | Add dome/key light; confirm RTX / PathTracing settings |
| Hang on stage load or first render | MDL compile or oversized stage | Follow the isolation sections below |
| Hang on shutdown | Native teardown or thread join may be blocked | Preserve logs and use external controls for the owned process; see Section 1a |

Diagnosis and resolution for Isaac Sim 6.0 (Kit 110) hangs, freezes, and perf degradation on large USD scenes (10K-150K+ prims, factory/warehouse twins).

Upstream debugging docs: `docs/isaacsim/utilities/debugging/` (profiling, Python debugging tutorials) in the pinned source tree.

## Quick Diagnosis Flowchart

```
Isaac Sim hangs/freezes
    |
    +-- During startup (before stage load)?
    |       -> Section 1: Startup Hangs
    |
    +-- During shutdown / SimulationApp.close()?
    |       -> Section 1a: Shutdown Hangs (Kit Framework Deadlock)
    |
    +-- During stage open / USD loading?
    |       -> Section 2: Stage Loading Hangs
    |
    +-- During shader/material compilation?
    |       -> Section 3: MDL/Shader Compilation
    |
    +-- During physics stepping / `SimulationManager` setup or timeline play?
    |       -> Section 4: Physics Hangs
    |
    +-- During Replicator / sensor capture?
    |       -> Section 5: Replicator Hangs
    |
    +-- During rendering (viewport frozen)?
    |       -> Section 6: Hydra/Rendering Hangs
    |
    +-- Nucleus/remote asset fetch?
    |       -> Section 7: Nucleus Network Hangs
    |
    +-- OOM crash or memory spike?
    |       -> Section 8: Memory Issues
    |
    +-- Slow but not frozen?
            -> Section 9: Performance Optimization
```

## Section 1: Startup Hangs

### Common Causes and Fixes

Kit startup flags shown upstream as `./isaac-sim.sh <args>` are Kit arguments; in Antioch pass them through `SimulationConfig.extra_args` (see [Isaac Sim on Antioch](../SKILL.md)).

**1. Extension conflict (circular import)**
```
--/app/extensions/exclude='["problematic.extension"]'
```

**2. Renderer initialization freeze**
```
--vulkan         # Force Vulkan
--reset-user     # Reset user settings
```

**3. Thread over-subscription on high-core-count CPUs**
```
--/plugins/carb.tasking.plugin/threadCount=16
--/plugins/omni.tbb.globalcontrol/maxThreadCount=16
```

**4. Startup hangs in the managed service** — collect the session startup logs and first error and keep them with the run record; do not kill other processes on the host.

## Section 1a: Shutdown hangs

If close never returns, retain the last logs and, when available, a native
stack trace. Plugin teardown or native thread joins can block even after the
task's Python code finishes. Do not infer the cause from the hang alone.

A Python thread timer or Python signal handler is not a reliable watchdog for
a native call that never returns to the interpreter. Python defers signal
handlers; see [CPython signal execution](https://docs.python.org/3/library/signal.html#execution-of-python-signal-handlers).

Use the platform's external process/session controls for the exact owned
operation. In Jupyter, interrupt may not stop native code; inspect state and
restart or stop that kernel when needed. Preserve outputs before termination
when possible, and report lost or unconfirmed finalization separately.

## Section 2: Stage Loading Hangs

### Diagnosis

Check layer count:
```python
for layer in stage.GetUsedLayers():
    print(f"  {layer.GetDisplayName()} ({layer.GetFileFormat().formatId})")
```
Many layers can increase resolution and load time. Measure before choosing a packaging change.

### Common Causes

**1. Excessive layer count (>1,000 layers)**

| Strategy | Cold Load | Layers |
|----------|-----------|--------|
| Per-asset files | 4 min | 11,488 |
| Library packaging | 53s | 8 |

Fix: Package assets into library layers (consolidate per-asset files into a small number of library `.usd`/`.usdc` layers; see [usd-composition-architecture](usd-composition-architecture.md) for the layered-asset pattern).

**2. Missing/broken references** — Cause USD to try every resolver including network fallbacks. Fix: Audit references:
```python
from pxr import Ar, Sdf

for layer in stage.GetUsedLayers():
    for ref in layer.GetExternalReferences():
        anchored = Sdf.ComputeAssetPathRelativeToLayer(layer, ref)
        if not Ar.GetResolver().Resolve(anchored):
            print(f"UNRESOLVED: {anchored}")
```

## Section 3: Shader/MDL Compilation

- First launch in a fresh environment compiles MDL shaders — can take 5-15 minutes, normal
- Subsequent launches use the shader cache
- Locate the active cache from startup settings/logs; paths depend on the engine image
- Cache lifecycle is managed service-side; report persistent shader-compile stalls with the session logs rather than deleting cache folders

## Section 4: Physics Hangs

**Physics setup / first-play hangs** (`SimulationManager.setup_simulation`, `timeline.play()`, or the legacy `world.reset()` flow — see [Renaming Extensions](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_4_5/extensions_renaming.html) to migrate off `omni.isaac.core.World`): usually caused by:
- `PhysicsScene` not yet defined when physics starts.
- Too many contact pairs on the first step.
- GPU dynamics enabled with too many rigid bodies (>100K).

Fix:
```python
# Ensure PhysicsScene exists before reset
if not stage.GetPrimAtPath("/World/PhysicsScene"):
    ps = UsdPhysics.Scene.Define(stage, "/World/PhysicsScene")
    
# Reduce initial contact storm
px_scene.CreateEnableStabilizationAttr().Set(True)
```

## Section 5: Replicator Hangs

- `rep.orchestrator.run()` hangs: ensure the timeline is playing (`omni.timeline.get_timeline_interface().play()` or `isaacsim.core.experimental.utils.app.play(commit=True)`) before invoking.
- `step_async()` never completes: use `await rep.orchestrator.step_async()` correctly inside an async context.
- Frame capture hangs: call `app_utils.update_app()` (or a `simulation_app.update()`) once before capture so the renderer has a fresh frame.

## Section 6: Rendering (Hydra) Hangs

- Viewport black and frozen: check GPU memory of the remote session
- RTX renderer OOM: Reduce texture streaming budget:
```python
carb.settings.get_settings().set("/rtx/resourcemanager/textureMipCountBudget", 256)
```

## Section 7: Nucleus Network Hangs

- Set timeout: `carb.settings.get_settings().set("/omni/client/timeout_seconds", 10.0)`
- Prefer assets bundled in the image or cached in the project over repeated remote fetches
- For fully offline work, pass `--/omni/client/enabled=false` via `SimulationConfig.extra_args`

## Section 8: Memory Issues

- Prims > 150K: Enable instancing (`UsdGeom.PointInstancer`)
- Textures: Use texture atlases, reduce resolution
- Monitor the session's GPU memory across the run to spot leaks

## Section 9: Performance Optimization

- Layer count: Package into library layers (see Section 2)
- Mesh instancing: Use `PointInstancer` for repeated assets
- Payload loading: Mark large sub-scenes as unloaded payloads
- Physics: Reduce collision mesh complexity for non-critical objects
