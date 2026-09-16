# Quality criteria

Adapted from NVIDIA [`isaac-sim-validator/references/quality-criteria.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-validator/references/quality-criteria.md) (Apache-2.0).

Select criteria before the run. The [validation workflow](isaac-sim-validator.md)
explains the evidence levels; [scenario design](../../scenario-design/SKILL.md)
records measured verdicts. These are review categories, not fixed thresholds.

| Deliverable | Evidence to require |
|---|---|
| Script or notebook | Import-safe saved source, correct startup owner, bounded execution, no unhandled errors |
| USD asset or scene | Resolvable dependencies, correct scale/frames, intended collision and rigid-body hierarchy |
| Physical behavior | Live state, relevant contacts/forces, task tolerances, repeatable initial conditions |
| Image | Decoded pixels, expected dimensions and viewpoint, fresh content, task-relevant visibility and appearance |
| Video | Ordered timestamps, continuous task coverage, inspected transitions and outcome |
| Dataset | Labels and sensor metadata aligned with frames, complete outputs, reproducible inputs |
| Learned policy | Reloadable checkpoint, learning metrics, bounded evaluation, required repeated or held-out cases |

Resource use and latency belong to the rubric when they are part of the task.
No universal image size, light intensity, training seed, checkpoint filename,
or warm-up count establishes quality across these workloads.
