# Rendering — Isaac Sim 6.0.1 on Antioch

Detail reference for the `isaac-sim-6` skill.

Everything here runs on a remote Antioch GPU machine. A streamed run shows the
Isaac GUI in the browser, but the stage does not run locally. Treat captured
frames and downloaded artifacts as the durable verification. The
boot/step/readback ordering and the 6.0 namespace map live in this skill's
`SKILL.md` — load it before writing any of this code. Dispatch and artifact
download live in `antioch-platform`; this reference covers only what runs
inside the scenario.

The scenario runner owns the app boundary: never construct `SimulationApp` in
scenario code, never call `antioch.boot()` in a scenario body (the runner has
already booted Kit), and never reference `.kit` configs, launch flags, or a
local install. Every pattern below is a loop *after* boot, inside the one
process a run gets.

## Capture pipeline — Replicator render product + annotators

The canonical capture path is a USD camera prim plus a Replicator render
product. Do not use `isaacsim.sensors.camera.Camera` for beauty renders — that
is sensor simulation and belongs to the sensors reference
(`references/sensors.md`).

```python
def make_capture(stage_path: str, resolution: tuple[int, int] = (1920, 1080)):
    import omni.replicator.core as rep
    from pxr import Gf, UsdGeom

    import antioch

    stage = antioch.stage()
    cam = UsdGeom.Camera.Define(stage, stage_path)
    xf = UsdGeom.Xformable(cam.GetPrim())
    xf.AddTranslateOp().Set(Gf.Vec3d(4.0, -4.0, 3.0))
    cam.CreateFocalLengthAttr().Set(20.0)

    render_product = rep.create.render_product(stage_path, resolution)
    rgb_annot = rep.AnnotatorRegistry.get_annotator("rgb")
    rgb_annot.attach([render_product])
    return render_product, rgb_annot


def capture_frame(rgb_annot, settle_frames: int = 200):
    """Render after an initial RTX-history budget; tune with frame stats."""
    import isaacsim.core.experimental.utils.app as app_utils
    import omni.replicator.core as rep

    for _ in range(settle_frames):  # initial budget; see the convergence rule below
        app_utils.update_app()
    rep.orchestrator.step()
    data = rgb_annot.get_data()  # (H, W, 4) uint8 RGBA
    return data[:, :, :3]
```

AOV annotators attach the same way — swap `"rgb"` for `distance_to_camera`,
`normals`, `semantic_segmentation`, `instance_id_segmentation`,
`motion_vectors`, and so on (full catalog, dtypes, and shapes:
`research_search` the annotator registry).

Publish every capture with `run.add_artifact(path)` so it lands on the
scenario's saved artifacts; download and review them with `antioch scenario download SCENARIO_RUN_ID`.
Full capture code (saving PNGs, AOV dtype handling, per-shot cameras) lives in
`references/rendering-capture.md`.

## RaytracedLighting vs PathTracing

```python
def set_render_mode() -> None:
    import carb

    carb.settings.get_settings().set("/rtx/rendermode", "RaytracedLighting")
    # carb.settings.get_settings().set("/rtx/rendermode", "PathTracing")
```

Mode strings are exact-match and 6.0.1 has one spelling everywhere:
`RaytracedLighting` (lowercase t) — the `/rtx/rendermode` setting,
`SimulationApp`'s `renderer` kwarg, and Kit's registered renderer list all
use it. `RayTracedLighting` (capital T) is pre-6.0 spelling from older docs
and upstream examples; port it on sight.

| Mode | Convergence | Cost | Use for |
|---|---|---|---|
| RaytracedLighting | start around 100–200, then probe | seconds per frame | Everything: iteration, interiors, video |
| PathTracing | converges over many subframes | 5–30 min per frame | Final hero shots only, when explicitly requested |

Default to RaytracedLighting. Switch to PathTracing only after lighting and
tone mapping are calibrated and the user asks for hero quality — PathTracing
for iterative work kills velocity.

## Lighting is mandatory

An authored stage has no default light, so captured frames are pure black
(RGB = 0) without one. Start with a baseline `DomeLight` + `DistantLight`
unless the scene recipe says otherwise
— the deep-occlusion recipe below is validated with no dome light. Baseline
code and per-scene intensity tables: `references/rendering-lighting.md`.

### ACES tone mapping — the single biggest quality lever

Without ACES, no amount of intensity tuning balances an enclosed render. This
is the most impactful setting after the lights themselves. Configuration code
and filmIso calibration: `references/rendering-lighting.md`.

Anti-recipes — do not waste runs on these:

- Wide rect lights (width 5+) → flat, no light pools.
- High dome intensity (400+) combined with filmIso 600 → washed-out shadows.
- Reinhard tone mapping → muddy, low contrast.
- PathTracing for iteration → minutes per frame.

### Enclosed interiors: multi-layer lighting

A ground-level camera in a narrow enclosed aisle sees a black frame
(mean RGB < 5) when only ceiling lights exist — RaytracedLighting struggles
with deep occlusion. The fix is layers: a ceiling rect-light grid for coverage,
plus sphere lights at head height inside each aisle, plus fog for depth.
Validated intensities at ACES filmIso 600, with no dome light:

| Layer | Light | Intensity | Shape / placement |
|---|---|---|---|
| Ceiling grid (dense, e.g. 8×14) | RectLight | 70,000 | 2.5 × 1.5 m panels, warm white (1.0, 0.97, 0.92), pointing down |
| Per-aisle fill | SphereLight | 15,000 | radius 0.1, at Z ≈ 3.5 m (head height), between shelf tiers |
| Ambient | none | — | ACES handles exposure; a dome washes out open views |

Result: mean RGB 60–155 across hero / overview / cross-aisle views. Full code,
fog settings, and the dome-vs-aisle exposure tension are in
`references/rendering-lighting.md`. Camera tip: put hero cameras at
intersections, not deep in narrow aisles — junctions let light reach.

## Settle frames before capture

Each `update_app()` in this loop advances Kit's renderer. The discarded frames
let the RTX denoiser and temporal accumulation history converge (and let
asynchronous materials finish); this is a render-history warm-up, not a
physics settling interval. Use a frame-statistics probe and stop when mean,
standard deviation, and highlights change by less than roughly 5% for three
consecutive frames. As a defensible starting budget, try 100–200 updates for a
normal/outdoor shot and 300–500 for a deeply occluded or path-traced interior,
then adjust for the scene and resolution. The old “200”/“500” values are
initial budgets, not universal constants.

Skipping render-history warm-up is a common cause of black or partial frames
after missing lights.

## Frame-quality validation — the remote QA gate

There is no screen to look at, so validate every frame numerically before
publishing it as an artifact. Never ship an unvalidated frame.

| Indicator | Meaning | Action |
|---|---|---|
| PNG ~82 KB | Black frame | Add explicit lights |
| PNG 200–500 KB | Partial render or trivial scene | Check settle frames |
| PNG 1–2 MB | Fully rendered frame | OK |
| `rgb.max() == 0` | No light reaches the camera | Add DomeLight + DistantLight |
| `rgb.mean() < 10` | Underexposed | Add low-level fill lights or raise filmIso |
| `rgb.std() < 5` | Flat / degenerate frame (no scene content) | Check camera points at the scene |
| `rgb.mean() > 220` | Overexposed | Reduce light intensity or filmIso |
| `rgb.max() > 200` and mean 60–180 | Good render | OK |

The validator code and the file-size + variance variants (for validating
downloaded artifacts locally) are in `references/rendering-capture.md`.

## Look-at camera math

Prefer a look-at matrix over hand-tuned Euler angles — the angles are brittle
and waste runs on sign flips. Matrix code, application via a matrix xform op
on the camera prim, third-person offset tables (behind/right/left of a
+X-facing robot), and dynamic chase-camera height for cluttered scenes are in
`references/rendering-capture.md`.

## Multi-episode / multi-shot batch loop

Antioch's warm machine plus one process per run is the persistent session —
do NOT re-dispatch per episode, and do NOT create a second `SimulationApp`.
Loop inside the scenario after boot: switch stages and reset simulation in
place, capturing per shot. Between episodes: clear the stage, garbage-collect,
and prefer instanceable assets for repeated geometry — Kit leaks GPU memory
when stages are not cleared, and a long batch run will OOM. The full loop
(ordering, memory discipline, throughput notes) is
`references/rendering-batch.md`.

## Video assembly

Frames come back as numbered PNG artifacts. Assemble locally after
`antioch scenario download SCENARIO_RUN_ID` (or on the machine before
publishing one mp4 artifact). Frame numbering must be gapless — ffmpeg silently stops at the
first gap, so name frames with a zero-padded global counter, not a per-episode
counter. Command and trade-offs: `references/rendering-capture.md`.

## Gotchas (symptom → cause → fix)

| Symptom | Cause | Fix |
|---|---|---|
| Pure black frame, PNG ~82 KB | No explicit lights | DomeLight + DistantLight baseline |
| Black frame despite ceiling lights | Deep-occlusion aisle, RaytracedLighting can't reach | Head-height sphere lights + a 300–500-update warm-up starting budget, then the stability probe |
| First frames dark/partial, later fine | Settle skipped | 200 frames (500 interiors) before capture |
| Washed-out open views | Dome too strong for filmIso | Drop dome; let ACES + local lights expose |
| Overexposed everything (mean > 220) | Intensity or filmIso too high | Lower intensity first, then filmIso |
| Flat gray frame (std < 5) | Camera inside geometry or aimed wrong | Look-at matrix; check camera pose numerically |
| ffmpeg video truncated | Gap in frame numbering | Zero-padded global counter |
| Batch run OOMs after N episodes | Stage not cleared between episodes | Clear + gc per episode; instanceable assets |
| `omni.isaac.*` import error | Removed in Isaac Sim 6.0 | Namespace map in `SKILL.md` |
| Capture empty despite annotator attached | No render step before `get_data()` | `app_utils.update_app()` then `rep.orchestrator.step()` |

## References (load one level deep when needed)

- `references/rendering-capture.md` — full capture code: camera prim
  setup, AOV annotator dtypes and post-processing, per-shot capture loop,
  chase-camera offsets and dynamic height, the complete frame validator
  (in-run and on downloaded files). Load when writing or debugging capture
  code.
- `references/rendering-lighting.md` — complete lighting code: baseline setups,
  ACES configuration, the multi-layer interior recipe with fog, exposure
  tension between view types. Load when tuning lighting or tone mapping.
- `references/rendering-batch.md` — the full multi-episode loop with memory discipline,
  instanceable assets, and headless throughput notes. Load when running more
  than one episode or shot per scenario.
