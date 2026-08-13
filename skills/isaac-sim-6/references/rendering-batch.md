# Batch rendering: multi-episode and multi-shot loops

The Antioch batch model: one scenario run gets one process on a warm GPU
machine. Warmth — Kit boot, shader compile, image pull — is already paid by
the machine. The remaining rule is: **pay the per-episode cost inside the
process, never re-dispatch per episode.**

- Do not construct `SimulationApp` in scenario code — the scenario runner owns
  the app boundary (`antioch.boot()`), and a second app corrupts Kit state.
- Do not split episodes into separate `antioch scenario run` invocations —
  that repeats dispatch and boot for every episode.
- Do loop over episodes and shots inside one scenario, switching stages and
  resetting simulation in place. This loop replaces the "persistent session"
  pattern from local-install workflows.

## Canonical batch loop

Current 6.0 APIs — the legacy `isaacsim.core.api.World` /
`isaacsim.core.utils.stage` paths are deprecated (see `SKILL.md`); this
loop uses the experimental/SimulationManager path because it survives
per-episode stage switching.

```python
def run_batch(run, episodes: int, steps_per_episode: int, settle_frames: int = 200) -> None:
    import gc

    import isaacsim.core.experimental.utils.app as app_utils
    import isaacsim.core.experimental.utils.stage as stage_utils
    from isaacsim.core.simulation_manager import SimulationManager

    for episode in range(episodes):
        stage_utils.open_stage(f"/path/to/scene_{episode}.usd")
        while stage_utils.is_stage_loading():
            app_utils.update_app()
        SimulationManager.setup_simulation(dt=1.0 / 60.0)
        app_utils.play(commit=True)

        # RTX denoiser + temporal accumulation warm-up, not physics time. The
        # default is an initial budget: probe frame stats and tune it (start
        # 100–200 normally, 300–500 for deep occlusion/path tracing).
        for _ in range(settle_frames):
            app_utils.update_app()

        for step in range(steps_per_episode):
            app_utils.update_app()
            # capture + validate_frame + run.add_artifact per shot

        app_utils.stop()
        clear_episode_state()  # close stage, drop references
        gc.collect()
```

Key ordering points (full rationale in `isaac-sim-6`):

1. Wait on `is_stage_loading()` before touching the new stage.
2. `play(commit=True)` before stepping physics — app updates only advance the
   simulation while the timeline plays. Replicator capture
   (`orchestrator.step()`) renders and captures regardless of timeline state.
3. Render-history warm-up happens after `play`, before the first capture of the
   episode — denoiser/temporal convergence restarts per stage. It is not a
   substitute for physics steps.
4. `stop()` then clear before the next `open_stage`.

## Memory discipline between episodes

Kit leaks GPU memory when stages are not properly cleared. A batch that
renders fine for 3 episodes can OOM at episode 20.

- Close/clear the stage and drop all Python references to prims, annotators,
  and render products from the finished episode, then `gc.collect()`.
- Detach annotators and release render products per episode — recreating them
  per stage is cheap; holding them across stages is the leak.
- Use instanceable assets for repeated geometry (racks, boxes, fixtures):
  `prim.SetInstanceable(True)` on the reference payload. Instanceable copies
  share GPU memory; non-instanceable copies each pay full cost.
- Watch GPU memory across the batch. Capture per-episode VRAM into
  `run.add_result("episode_vram_mb", ...)` so a growth trend is visible on the
  saved run results instead of surfacing as a mid-batch CUDA OOM.

```python
def gpu_memory_mb() -> float:
    """Device-wide used VRAM — torch.cuda.memory_allocated() only sees torch's
    caching allocator, not the Kit/RTX/Hydra allocations that leak here."""
    import torch

    free, total = torch.cuda.mem_get_info()
    return (total - free) / 1e6
```

Import `torch` inside the function and only after physics has settled —
importing it early can hang Kit (CUDA context conflict; see `isaac-sim-6`).

## Throughput notes

- RaytracedLighting warm-up dominates shot cost: start with 100–200 updates
  (300–500 for occluded interiors), then use a frame-statistics stability probe
  rather than treating a count as a contract. Per-shot captures are cheap if
  the camera moves but the lighting does not.
- Rendering off (physics-only episodes) steps at 10–50× real-time — if an
  episode needs no frames, step with `SimulationManager.step()` (no app
  update, no render) and skip settle entirely.
- First step on a cold stage can take minutes (MDL shader compile). Warm
  machines pre-warm this, but budget for it on new scenes — a slow first step
  is not a hang.
- Long batches belong to a queued suite (`antioch suite run NAME --queue`,
  followed with `antioch suite show SUITE_RUN_ID --follow`) rather than a
  blocking `antioch scenario run`. Build the project source into the sim image
  first; queue workers do not apply development watch rules. See
  `antioch-platform`.
