# Synthetic data and MobilityGen

Use this for annotated frames, domain randomization, or mobile-robot
record/replay datasets. Sensor setup is in `references/sensors.md`; capture
and cleanup are in `references/rendering.md`.

## Choose the pipeline

- Replicator captures scenes with camera/object/light randomization and
  annotators.
- MobilityGen records robot trajectories, then replays them with sensors.
- Physical navigation or manipulation evaluation needs its own behavioral
  verdict; a rendered dataset is not that verdict.

Define dataset schema, modalities, labels, calibration, seeds, frame count,
and validation before scaling up. Start with a small representative sample
and inspect the saved outputs through the intended reader.

## Replicator

The runtime entry point remains `omni.replicator.core`. Use the pinned writer
registry and inspect the chosen writer's initialization parameters; writers
do not all share a schema. Common choices include `BasicWriter` and dataset-
specific writers. Retrieve support for specialized outputs such as pose or
Cosmos data instead of assuming an installed extension is active.

Label the prims that should appear in semantic and bounding-box outputs.
Check taxonomy and label propagation to descendants/instances. Empty boxes
can mean missing semantics, occlusion, or no object in view.

For manual capture:

1. Disable automatic capture-on-play when the workflow should own captures.
2. Set or randomize scene state through the chosen authoring API.
3. Complete `rep.orchestrator.step` or await `step_async` with the requested
   simulated-time behavior. A zero `delta_time` is useful for static captures.
4. Read the required annotators only after their capture completes.
5. Drain `rep.backends.io_queue.wait_until_done()` before inspecting or
   archiving file-writer output.
6. Detach writers/annotators and destroy owned render products on success
   and failure.

`rep.orchestrator.run()` schedules work; returning from it alone does not
prove the dataset is complete. Use the synchronization API appropriate to
the chosen synchronous or asynchronous workflow.

Check each annotator's dtype, shape, units, metadata, and invalid-value
convention. Raw RGB annotators can include alpha. Depth's background values
may be invalid by design; validate finite positive depth on the relevant
mask, not every background pixel.

## Randomization and reproducibility

Feed explicit seeds to each random source actually used, including NumPy,
Replicator, task code, and any external library. Record the image/asset pins
and generation parameters. The same seed and engine version do not guarantee
bit-identical GPU physics or renderer output.

Choose randomization distributions from the training/evaluation goal. Check
collisions, visibility, class balance, and calibration after randomization.
Tuning capture subframes can reduce temporal artifacts; measure the result
instead of copying a fixed count from another scene.

## MobilityGen record and replay

The pinned API is `isaacsim.replicator.experimental.mobility_gen`; example
robots and scenarios live under `isaacsim.replicator.mobility_gen.examples`.
Enable the needed extensions before importing them. Retrieve the pinned
recording and replay examples together, including their setup and teardown.
Do not assemble a "complete" pipeline from placeholder calls.

Record phase:

- Load the exact scene and matching occupancy map.
- Build the robot and scenario with the documented simulation lifecycle.
- Reset before each episode and record the initial/step state required by the
  reader.
- Keep scene dependencies, robot configuration, sensor overrides, map, and
  trajectory data together.
- Close/flush the writer even when a scenario ends early.

Replay phase:

- Register custom robot/scenario classes before resolving recorded type names.
  Keep their simulator imports inside a factory called after Kit startup.
- Load the recorded scene and matching robot definition. A type name alone
  does not pin mutable class attributes such as `physics_dt`.
- After enabling the required rendering modalities, call
  `scenario.finalize_rendering()`. The pinned standalone replay then starts
  its timeline and updates the app before applying sensor overrides, so the
  render graph exists. Do not replace that initialization with an extra
  orchestrator capture.
- Apply recorded state and synchronize it through the documented replay path
  before capture. Do not add unrelated physics steps that change the recorded
  state.
- Complete sensor capture with `rep.orchestrator.wait_until_complete()` and
  drain output. Stop the replay-owned timeline before
  `scenario.disable_rendering()` or loading the next scene; the pinned example
  warns that a playing timeline across teardown can crash Kit. Close the
  writer and retain owned-resource cleanup on failure too.
- Validate frame/state alignment, calibration, modality counts, and expected
  output decoding.

Keep record and replay in one run when that fits the dataset. Separate them
when re-rendering saved trajectories is a requirement, and use durable
artifacts for the handoff. The pinned `replay_directory.py` defaults to output
that links back to its source recording, not a standalone dataset. Preserve
that source and its dependencies, or use the script's `--self_contained`
option before archiving the output; verify the archive in an isolated path.
A live GUI can support interactive workflows, but
keyboard/gamepad examples require the corresponding input path; do not assume
that a headless batch has one.

## Artifact and validation contract

Use an owned scratch directory. Session files are temporary. Archive the
completed dataset and call `run.add_artifact`; save validation measurements
with `run.add_result` and verdicts with `run.check`.

Validate each requested modality and camera separately. Counting all PNGs
together can pass when RGB is missing but segmentation files exist.
Match frame IDs across modalities, decode samples, check required labels and
box areas, and test calibration against known geometry. Tiny file sizes do
not prove blank frames; large files do not prove useful content.

Scale by cases or episode shards when independent outcomes and retries are
useful. Review the requested scale before starting a costly sweep. Report
requested, completed, failed, and retained frames; keep diagnostic samples on
failure and never describe a partial dataset as complete.
