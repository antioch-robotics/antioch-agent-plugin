# Isaac troubleshooting

Start with the failing run's status, logs, and artifacts through the platform
skill. Keep the engine/image identity, first error, input, and smallest
reproduction. A symptom suggests checks; it does not uniquely identify a cause.

| Symptom | First checks |
|---|---|
| Local discovery import failure | Simulator import at module scope; transitive imports count too |
| Missing vendor symbol | Exact module and runtime pin; retrieve exports and callers |
| First reset/step stalls | Asset loading, shader/kernel compilation, physics initialization, initial overlaps, process stack |
| Body appears stationary | Live pose versus stale USD/cache, timeline progress, kinematic flag, drive/state command |
| Sensor data absent or stale | Initialization, render/physics cadence, timestamp, buffer lifetime, completion |
| Unexpected motion | Units, quaternion ordering, batch/joint indices, active backend, collision geometry |
| Collision or grasp fails | Cooked collider, filters, contact reporting, limits, payload and initial penetration |
| Black or white output | Capture readiness, camera, clipping, visibility, lights/materials/exposure; inspect decoded pixels |
| Dataset empty or partial | Per-modality writer output and completion/flush, not merely a scheduled coroutine |
| Memory grows across shots | Owned render products, annotators, writer queues, arrays and stage lifetime |
| Newton API/shape mismatch | Standalone solver versus USD integration, actual backend, supported tensor layout |
| Run claims success with wrong result | Missing task oracle, last-sample-only check, or simplified model hiding failure |

Use a bounded minimal probe to isolate one boundary. Profile before changing
thread counts, solver settings, or GPU budgets. Do not infer a permanent
engine limitation from a single scene, then replace physical motion with an
analytical script.

For a stalled process, collect available logs and a stack before stopping it.
Stop only the process/session in scope. Repeating an identical failing
dispatch without new evidence is not a diagnosis.

A warning is benign only when the relevant behavior is verified. Keep unknown
warnings with the result; do not suppress them to make a run look clean.

Report separately: source-verified API, observed runtime behavior, failed
checks, and remaining hypotheses. Hardware behavior is unverified until the
corresponding runtime probe actually runs.
