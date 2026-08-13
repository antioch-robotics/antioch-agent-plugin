# Replicator static-scene SDG — depth

Load this when building or debugging the static-scene pipeline. Everything here
runs inside an Antioch scenario body on Isaac Sim 6.0.1 / Kit 110.1.2 — imports
stay inside function bodies (Rule #1), outputs go to a scratch dir and leave as
one archived artifact (`run.add_artifact`), and validation numbers leave as
`run.add_result(...)`.

When a dataset camera is created through `rep.create.render_product`, use the
single resolution-order table in `references/sensors-cameras.md`; this SDG
reference does not restate the `(width, height)` versus `(height, width)` rule.

## Writer initialization

Two equivalent forms. Prefer the explicit-backend form when several writers
share one output directory:

```python
backend = rep.backends.get("DiskBackend")
backend.initialize(output_dir=str(output_dir))
writer = rep.writers.get("BasicWriter")
writer.initialize(backend=backend, rgb=True, bounding_box_2d_tight=True)
writer.attach(render_product)
```

Short form (backend created implicitly from `output_dir`):

```python
writer = rep.WriterRegistry.get("BasicWriter")
writer.initialize(output_dir=str(output_dir), rgb=True, bounding_box_2d_tight=True)
writer.attach(render_product)
```

- `rep.WriterRegistry.get(name)` and `rep.writers.get(name)` read the same
  registry.
- `FPSWriter` records per-frame capture time — attach it during throughput
  tuning, drop it for production runs.
- Custom writer: subclass `omni.replicator.core.Writer` for direct access to
  annotator tensors, register with `rep.writers.register_writer(...)`.
- `CosmosWriter` needs `carb.settings.get_settings().set("/app/omni.graph.scriptnode/opt_in", True)`
  before the first capture, and writes PNG sequences plus one MP4 per modality
  (rgb, shaded_seg, segmentation, depth, edges).
- Deprecated, never emit: `DOPEWriter`, `YCBVideoWriter`, `PytorchWriter`,
  `PytorchListener` (and the `OgnPose` node).

## Annotator catalog

Via `rep.annotators.get(name)` or
`rep.AnnotatorRegistry.get_annotator(name, init_params={...})`. The full
catalog of names, return dtypes, and shapes: `research_search` it. Three
traps worth knowing up front: `rgb` returns RGBA uint8 `(H, W, 4)` — drop
the alpha channel for datasets; `distance_to_image_plane` can contain NaN;
semantic and bbox annotators take `init_params={"semanticTypes": ["class"]}`
to restrict which semantics they read, and `semantic_segmentation` takes
`{"colorize": False}` for raw ID maps instead of colors.

## Reading annotators directly (no writer)

Writers are fire-and-forget behind a background backend. When the scenario
itself must validate frame content (brightness, labels, box areas) before
deciding the run passed, read annotators synchronously with per-frame deadlines
— this pattern is e2e-verified on Antioch:

```python
def capture_checked(run: "antioch.ScenarioRun", camera: object, frames: int) -> None:
    import time

    import numpy as np
    import omni.kit.app
    import omni.kit.async_engine
    import omni.replicator.core as rep

    render_product = rep.create.render_product(camera, (640, 480), name="checked_view")
    rgb = rep.AnnotatorRegistry.get_annotator("rgb")
    rgb.attach(render_product)
    app = omni.kit.app.get_app()
    deadline = time.monotonic() + 15.0
    for frame in range(frames):
        capture = omni.kit.async_engine.run_coroutine(rep.orchestrator.step_async(rt_subframes=4, delta_time=1.0 / 60.0, wait_for_render=True))
        while not capture.done():
            if time.monotonic() >= deadline:
                capture.cancel()
                rep.orchestrator.stop()
                raise RuntimeError(f"capture {frame + 1}/{frames} exceeded the 15s deadline")
            app.update()
        capture.result()
    rgba = np.asarray(rgb.get_data())
    assert rgba.shape == (480, 640, 4), f"rgb annotator returned shape {rgba.shape!r}"
    assert float(rgba[:, :, :3].mean()) > 1.0, "rgb frame is black — check lights and camera pose"
```

Pumping `app.update()` between polls keeps a stalled renderer cancelable — the
run dies on the deadline instead of hanging to the CLI timeout.

## Semantics: label prims or annotators see nothing

Both forms write the same `UsdSemantics.LabelsAPI` schema:

```python
add_labels(prim, labels=["cardbox"], taxonomy="class")  # isaacsim.core.experimental.utils.semantics
rep.functional.modify.semantics(prim, {"class": label}, mode="add")  # equivalent functional form
```

`rep.create.*` primitives accept semantics inline:
`rep.create.cube(..., semantics={"class": "blue_crate"})`.

## Domain randomization with `rep.functional`

Call between `rep.orchestrator.step(...)` captures, driven by the scenario's
seeded `np.random.Generator`:

```python
# Scatter objects on a surface with collision checks
rep.functional.randomizer.scatter_2d(prims=objects, surface_prims=[plane], check_for_collisions=True, rng=rng)

# Camera or object pose (rotation_value is a quaternion, WXYZ)
rep.functional.modify.pose(camera, position_value=rng.uniform(pos_min, pos_max).tolist(), look_at_value=target_prim, look_at_up_axis=(0, 0, 1))
rep.functional.modify.pose(
    prim,
    position_value=(rng.uniform(-15, 5), rng.uniform(-5, 10), 0),
    rotation_value=euler_angles_to_quaternion([0, 0, rng.uniform(0, 2 * math.pi)]).numpy().tolist(),
)

# Lights via OmniGraph events authored on the stage
rep.utils.send_og_event(event_name="randomize_lights")
```

`euler_angles_to_quaternion` comes from
`isaacsim.core.experimental.utils.transform`. Re-randomize object poses every
few frames rather than every frame — pose deltas between consecutive frames are
what force high `rt_subframes`.

## Throughput knobs

| Knob | Effect |
|---|---|
| `rt_subframes` per step | re-renders the same frame to kill ghosting from pose deltas and converge materials; 4–8 for RTX real-time + DLSS Quality, 16–32 for path tracing or heavy material streaming |
| `rep.orchestrator.step(wait_for_render=False)` | decouples capture from render completion — faster, but data may correspond to a previous frame; only when strict frame↔data correspondence is not required |
| `/exts/omni.replicator.core/enableWriteToFabric = True` | randomization deltas go straight to Fabric instead of through USD; faster, transient — not persisted to the stage |
| `rtx/post/dlss/execMode = 2` (DLSS Quality) | recommended for SDG; the default Performance mode produces edge artifacts below ~600×600 |
| resolution / modality count | the cheapest lever — bounding boxes cost far less than segmentation or depth |

Set carb settings once at scenario start:

```python
carb.settings.get_settings().set("rtx/post/dlss/execMode", 2)
carb.settings.get_settings().set("/exts/omni.replicator.core/enableWriteToFabric", True)
```

Measure before tuning: attach `FPSWriter` for one run and read the capture
times out of the archive.

## In-run output validation

Adapted from NVIDIA's `validate_sdg_output.sh` — run it against the writer's
output dir before archiving, and surface the numbers with `run.add_result`:

```python
def validate_writer_output(output_dir: Path, expected_frames: int) -> dict[str, int]:
    """Count and sanity-check one writer output directory before archiving it."""
    images = list(output_dir.glob("**/*.png")) + list(output_dir.glob("**/*.jpg"))
    annotations = list(output_dir.glob("**/*.json")) + list(output_dir.glob("**/*.npy"))
    empty = [path for path in images if path.stat().st_size < 1024]
    assert images, f"writer produced no images under {output_dir}"
    assert len(images) >= expected_frames, f"expected {expected_frames} frames, found {len(images)}"
    assert not empty, f"{len(empty)} suspiciously small images (likely black frames): {empty[0]}"
    return {"images": len(images), "annotation_files": len(annotations)}
```

Deeper checks (mean brightness, expected class IDs in segmentation maps,
non-zero box areas, no NaN in depth) need the direct-annotator pattern above —
file sizes alone cannot see content.

## Sibling SDG workflows (not this reference's core, same engine)

| Workflow | Module |
|---|---|
| Grasp dataset generation | `isaacsim.replicator.grasping` (`GraspingManager`, `GraspPhase`) |
| Episode record + replay | `isaacsim.replicator.episode_recorder` |
| Teleop record + replay | `isaacsim.replicator.teleop` (needs a UI — not headless) |
| USD-persistent randomization behavior scripts | `isaacsim.replicator.behavior` |
| Mobile-robot trajectory SDG | `references/sdg-mobility-gen.md` |
