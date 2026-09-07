# Rendering, capture, and image evidence

Use this for screenshots, video, lighting, camera framing, and batch rendering.
The requested image or dataset decides quality; a canned lighting recipe does
not.

## Start with the requested shot

Identify the subject, view, resolution, renderer, and output format.
Respect existing scene lighting and materials unless the task asks to change
them. For a new scene, a modest environment light and a directional key can
be a starting point, not a required pair or universal intensity.

Headless RTX supports MDL. If a frame is black, inspect material resolution
and shader logs as well as lighting, camera, clipping, visibility, renderer,
and capture readiness. A shader type alone is not a compatibility test.

Raw USD camera optical units differ from scene-distance units; see
`references/sensors.md` before setting focal length, aperture, or calibration.

## Capture lifecycle

1. Start through the parent skill's lifecycle and finish scene/camera setup.
2. Select the viewport camera or create a dedicated render product.
3. Attach the required annotator or writer.
4. Drive capture through the selected API and wait for render completion.
5. Read a fresh frame, inspect it, and retain useful evidence.
6. Flush file writers before checking or archiving their outputs.
7. In `finally` cleanup, detach owned resources and destroy only products
   created by this operation.

With Replicator, `rep.create.render_product` takes `(width, height)`.
`rep.orchestrator.step` is the synchronous capture path;
`step_async` must be awaited or driven by the appropriate Kit event loop.
Do not call `asyncio.run` inside an already-running loop or confuse scheduling
a coroutine with completing it.

Warm-up depends on asset loading, shader compilation, temporal accumulation,
and the selected renderer. Use readiness/content checks and a deadline.
There is no universal 100/200/500-frame requirement. A per-frame deadline must
be reset for each frame; an outer deadline is a whole-job budget.

## Read the image, not its file size

PNG size depends on resolution and compression. A valid small image can be
under 100 KB; a large black image can also compress very well. Neither byte
count nor a hash is a render-quality oracle.

Check the decoded payload first: nonempty dimensions, expected channels,
dtype/range, and finite numeric data where required. Then check the task:
subject visibility, framing, occlusion, geometry, semantic masks, or measured
features. Mean and variance can screen obviously degenerate frames but cannot
prove that the right scene is visible. Dark, white, or low-contrast scenes
can be intentional.

Keep a failed image with the failing measurement. Do not make a recording
look healthy by omitting the bad frames. Diagnostic evidence and accepted
deliverables can use separate paths.

## Diagnose before tuning

| Symptom | Check |
|---|---|
| No frame | Capture API return contract, initialization, rendering enabled, completion and timeout |
| Stale frame | Timestamp, camera selection, synchronization and buffer lifetime |
| Black frame | Camera pose/clipping, visible geometry, lights, unresolved assets/materials, renderer logs |
| White or washed out | Exposure, light energy, material response, tone mapping |
| No subject despite useful contrast | Projected subject bounds or semantic/known-feature pixels |
| Temporal ghosting | Motion/capture synchronization and renderer accumulation |
| Increasing GPU memory | Undetached annotators, retained buffers, per-shot products and writer queues |

Change one relevant variable at a time and compare saved frames. Do not
override tone mapping, disable physics, or switch to path tracing solely
because an unrelated recipe says to. Path tracing may improve a requested
shot but changes cost and convergence.

## Batch and video work

Reuse a process and render products when shots share a scene and lifecycle.
Reset task state explicitly between episodes and release per-episode buffers.
Separate Antioch cases when they need independent outcomes, seeds, retries,
or artifacts. A many-shot dataset can also be one case; neither layout is a
universal rule.

For video, define simulated time versus presentation time, frame cadence,
and dropped-frame policy. Count decoded frames and check timestamps; a file
that opens is not enough. Publish the completed video or frame archive through
the run artifact API.

Reduce resolution, modality count, or scene cost only within the requested
fidelity. Profile asset loading, render time, device transfers, encoding,
and writer flushing separately before optimizing.
