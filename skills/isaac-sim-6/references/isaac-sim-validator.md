# Isaac Sim validation

Adapted from NVIDIA [`isaac-sim-validator/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-validator/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup and import rules, and
[scenario design](../../scenario-design/SKILL.md) for recorded verdicts.

## Purpose and levels

Validate the requested deliverable, not a particular demo's size or appearance.
Separate four kinds of evidence:

1. **Static:** Python parses and project discovery succeeds without Isaac.
2. **Structural:** scene units, dependencies, schemas, frames, and simulator
   ownership match the intended model.
3. **Runtime:** a bounded remote execution produces valid state and outputs.
4. **Task:** measured behavior and inspected artifacts meet the user's criteria.

Static and structural checks do not establish physics or rendering correctness.
Executing a script is an action with effects, not a stronger static check.
Use the authorized Antioch session and a bounded experiment.

## Script and scene validation

- Check import safety, startup ownership, required extensions, and the selected
  engine. Deprecated does not mean absent; use the pinned interface.
- Check stage up-axis, units, transforms, referenced assets, materials, and
  visual versus collision geometry. A centimeter-authored robot can be valid
  when composition and conversion are correct.
- Keep intended rigid-body ownership and joint frames. Choose collision
  approximations and instancing for the task; neither has one universal setting.
- Deliver reusable paths and configuration. Runtime asset URLs may be valid;
  resolve all dependencies when exporting a portable asset.
- Define expected outputs and stopping conditions before execution. Script
  byte count and an arbitrary number of simulated seconds prove neither.

## Render validation

Decode the actual image. Check dimensions, channels, freshness, camera pose,
subject visibility, materials, exposure, and the requested appearance.
Inspect lighting, sensor readiness, and camera framing when output is black.

File size and RGB statistics are diagnostic clues, not universal pass/fail
thresholds. A simple red cube can compress well; darkness can be intentional.
Sensor render products need not match viewport dimensions. Light types,
tonemapping, and renderer choice depend on the scene and task.
See [rendering](isaac-sim-rendering.md) for tuning examples.

## Video validation

Inspect frames across the recording, including transitions and the task outcome.
Check timestamp order, continuity, framing, and the expected motion. Keep the
complete source evidence if a shorter presentation cut removes warm-up frames.
A smooth movie alone does not prove physical contact or successful control.

## RL training validation

Use [Isaac Lab](../../isaac-lab-3/SKILL.md) for the framework lifecycle and
training workflow. Check finite observations/actions, reset behavior, reward
and termination logic, learning metrics, and resource use. Load the saved
checkpoint and evaluate it; its filename or existence is not success.
Use repeated seeds and held-out cases when the claim requires generalization.

## Report and automation

Report failed checks and unrun levels separately. Preserve the input, source,
run identity, logs, and artifacts needed to reproduce a failure. Do not change
the physical task or lower a threshold to produce a pass.

The upstream [validation script](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-validator/scripts/validate_sim.sh)
is a source example, not an installed command or Antioch acceptance gate. Its
local-launch and demo-specific thresholds need adaptation. Prefer project
checks and recorded scenarios for reusable validation.

Use the [quality rubric](isaac-sim-validator-references-quality-criteria.md)
to select criteria for the deliverable.
