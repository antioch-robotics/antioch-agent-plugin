# Diagnose a failed workflow

Read the exact error and confirm the project, deployment, SDK, session, and
run identity before changing anything:

```bash
antioch version --json
antioch auth whoami --json
antioch session status --json
antioch service ps --json
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario logs SCENARIO_RUN_ID --json
```

Source, installed SDK, and deployed service can differ. Installed help and
server errors establish available behavior. A feature in a checkout is not
evidence of deployment; never switch identity to make a test pass.

| Symptom | Inspect first |
|---|---|
| Collection fails on native imports | Module-scope or transitive imports; defer them until after startup |
| Code looks stale | Source mappings, ignore rules, and the selected session; submitted runs use saved bundles, reruns use saved inputs |
| Dependency missing | Frozen image and extension availability; rebuild and start a new session if needed |
| Session not ready | Service process state and first startup error |
| Stream unavailable | Selected service and existing producer; use `--no-stream` if video is unnecessary |
| Scenario errored | Admission, startup, execution, or publication error; reproduce one bounded case |
| Scenario failed | Named checks and measured values |
| Viewer empty | Saved telemetry samples, entity paths, frames, and layout selectors |
| Asset unavailable | Exact name/version, scope, and transfer error |
| Timeout or lost connection | Existing operation state before retrying side effects |

For kernel recovery, read [Jupyter](../../agentic-simulation/references/jupyter.md).
For startup/rendering failures, read [Isaac troubleshooting](../../isaac-sim-6/references/isaac-sim-troubleshooting.md).

Collection validates definitions without running their bodies. Runtime
verification needs terminal records and relevant artifacts: inspect actual
pixels for visual claims and live body state for physical claims. Confirm
which source bundle and images ran. Retain failures, save remote outputs,
and release only compute owned by the task. Report unrun checks as unrun;
never equate submission or upload success with correct simulation behavior.

For code that automates these operations, read [Python errors and submission recovery](python-client.md#handle-failures-and-notices). Return to [Antioch platform](../SKILL.md#capability-guides) to choose the owning guide before changing configuration.
