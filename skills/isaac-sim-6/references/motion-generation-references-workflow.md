# Motion-generation demo workflow

Adapted from NVIDIA [`motion-generation/references/workflow.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/references/workflow.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

Use this sequence for new motion-generation demos. It assumes the generic
interactive-session, rendering, physics, and manipulation validation rules are
loaded from the sibling skills listed in [Isaac Sim on Antioch](../SKILL.md). The controller is
pluggable; for the cuMotion `RmpFlowController` specifics see
[cumotion](motion-generation-references-cumotion.md).

## Runtime choice

- **Interactive session**: default for iteration, stage inspection, marker
  placement, screenshots, and validation probes (Jupyter kernel in the Antioch
  session; see [agentic-simulation](../../agentic-simulation/SKILL.md)).
- **Standalone script / scenario**: deliverable shape when the demo must own
  startup, simulation timing, CLI args, and pass/fail reporting. A plain script
  calls `antioch.start_simulation()`; a managed scenario declares
  `SimulationConfig`. Choose the execution path that fits the requested task.
- **Interactive `BaseSample` extension**: use only after the core sequence is
  stable. Port setup/reset/callback structure, not old IK APIs.

## Demo checklist

1. Define task success as measured state transitions, not just tool motion.
2. Read the pinned motion-generation docs/example for the selected robot,
   plus the controller-specific reference (e.g. [cuMotion](motion-generation-references-cumotion.md)).
3. Build the smallest stage: robot, support surface, one object, lights, and
   only required obstacles.
4. Inspect robot DOFs, link paths, controller tool frames, object AABB,
   rigid-body state, collision APIs, and mass.
5. Add debug markers for the controller target, measured tool pose, physical
   grasp point, object center/grasp point, and relevant axes.
6. Define the axis contract: tool approach axis, object semantic axis, and
   desired final object axis.
7. Compute any tool-local grasp offset in the same orientation used for
   approach; freeze it while closing and lifting.
8. Build the world binding (`SceneQuery`, `ObstacleStrategy`, `WorldBinding`;
   see [world binding](motion-generation-references-world-binding.md)) and the motion-generation controller (cuMotion
   `RmpFlowController`; see [cuMotion](motion-generation-references-cumotion.md)).
9. Implement an explicit phase machine. Every phase needs:
   - target pose
   - controller reset behavior
   - convergence predicate
   - timeout
   - trace output with measured object/tool state
10. Reset the controller at discontinuous target jumps.
11. Validate gates in order with [manipulation-ik](manipulation-ik.md): pick-up/hold,
    manipulate/flip, and place/release. Do not continue after a failed gate.
12. Capture visual evidence from the latest run with the interactive-session
    helpers (see [agentic-simulation](../../agentic-simulation/SKILL.md)).
13. Keep reusable corrections with the project before delivery.

## Phase machine shape

Use named phases rather than hidden timers. Start with `PHASES` from
[`scripts/phase_machine.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/phase_machine.py) and remove phases that do
not apply to the task.

Motion phases should converge on measured state. A timeout in a motion phase is
a failed or incomplete phase, even if a final pose oracle happens to pass.
Fixed-frame completion is only acceptable for intentional dwell, hold-open, or
settle phases.

## Standalone scaffold

Use [`scripts/cumotion/standalone_demo_template.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/motion-generation/scripts/cumotion/standalone_demo_template.py)
as a starting point for standalone demos. It owns simulation startup and
the required post-startup imports.

Do not copy standalone startup calls into interactive-session probes. In
particular, `SimulationManager.setup_simulation(...)` belongs in scripts that
own startup. In a running session, stop/play with `app_utils`, reset or
recreate the stage, and step with `await app_utils.update_app_async()`.

## Finish criteria

- The phase machine reaches a terminal phase without unhandled exceptions.
- Motion phases complete by measured convergence, not by motion timeouts.
- Gate validation passes in order and includes object-state evidence.
- The latest run has fresh visual evidence from the interactive session.
- The result uses the requested physical interaction model. If the task is
  contact-only, no pose assist, hidden constraints, or direct object pose writes
  are used as the success path.
