---
name: agentic-simulation
version: "1.2.4"
description: The workflow entry point for agents using Antioch to research, design, build, inspect, test, and improve simulations. Use for Antioch simulation questions and development, custom kernels and solver integration, parameter fitting, datasets, policy experiments, failure diagnosis, and saved-run analysis. Connects research, native Python, CLI, Jupyter, assets, scenarios, and suites in a measured engineering loop scoped to the request. Load antioch-platform alongside it for the programming/cloud model, core concepts, and CLI/YAML workflows.
---

# Agentic simulation

Help the user turn a robotics or simulation question into grounded knowledge,
working code, or a measured result. Antioch connects research, user-owned Python,
remote GPU execution, and recorded evidence. Load
[Antioch platform](../antioch-platform/SKILL.md) for its programming/cloud model,
projects, services, sessions, scenarios, and suites.

Scale the work to the request. A question can end with a researched answer;
a small script need not become a suite or training project. Editing, dispatch,
data collection, and publishing must stay within the user's authorized scope.

## The development loop

Use only the steps needed for the request; research alone needs no project or compute.

1. **Define the result.** Identify the requested behavior and how to measure
   it: images for appearance, live state and contacts for physical behavior,
   repeated cases for reliability.
2. **Prepare.** Find or complete the owning project. Use
   [research](../antioch-research/SKILL.md) to choose methods and resolve
   interfaces across libraries. Load [Isaac Sim](../isaac-sim-6/SKILL.md) or
   [Isaac Lab](../isaac-lab-3/SKILL.md) for the selected runtime, and check
   [existing assets](../antioch-platform/references/assets.md). Implement the
   smallest useful change.
3. **Run a bounded experiment.** Choose the mechanism below. Record the
   source, inputs, and execution identity needed to interpret its output.
4. **Inspect and compare.** Read exceptions, measurements, and actual images.
   Preserve failures; a completed command is not a passing evaluation.
5. **Improve and retain.** Fix the cause supported by the evidence, sync or
   rebuild, and test again. Keep useful code, configurations, failed hypotheses,
   and validated results in project files and run artifacts for the next task.

Keep the requested physical model. Teleporting a robot, welding a payload,
or disabling collisions changes what the experiment proves. Use such
approximations only when they are part of the agreed task.

## Tools and workflows

| Work | Mechanism |
|---|---|
| Research methods, APIs, assets, and examples across libraries | [Research MCP](../antioch-research/SKILL.md): search, expand, open, grep, inspect versions |
| Set up or change compute, images, dependencies, and source | Platform [environment](../antioch-platform/references/environment.md), [manifest](../antioch-platform/references/manifest.md), and [session CLI](../antioch-platform/references/sessions.md) |
| A script with a finite lifetime | `antioch service exec python src/main.py` |
| Explore and modify a live scene | [Jupyter cells](references/jupyter.md) |
| Inspect a viewpoint or sensor camera | [Navigation and capture](references/viewport.md) |
| Repeatable checks over inputs | [Scenario design](../scenario-design/SKILL.md), then scenario or suite dispatch |
| Investigate previous experiments | Platform [scenario history](../antioch-platform/references/scenarios.md) and [suites](../antioch-platform/references/suites.md) |
| Generate datasets or develop policies | Isaac Sim [data collection](../isaac-sim-6/references/data-collection-sim.md) or [Isaac Lab](../isaac-lab-3/SKILL.md), then measured evaluation |

Preserve the user's choice of script, notebook, or recorded evaluation.
Move an experiment into a scenario when it needs repeatable verdicts, cases,
or durable comparison.

Simulation is also a design choice: combine native APIs, Newton, custom Warp
kernels or solvers, renderers, sensors, and controllers as the task requires.
Research each integration and test its coupled behavior; components working
separately do not prove they work together. Prefer existing assets and methods
before building replacements or training a new model.

When fitting to real measurements, record units, calibration, and adjustable
parameters; compare against a baseline and separate fitting data from validation
data. Retain inputs and errors with the experiment; use scenario results and
artifacts for recorded evaluations.
Simulation evidence does not establish real-world reliability without relevant
real-world validation.

## Close the loop with saved runs

Preview definitions with `antioch scenario collect --json` and
`antioch suite collect --json`. These expose parameters, cases, tags, and
source paths without requesting compute or executing scenario bodies.

Use CLI JSON to find and compare runs. Scenario history supports filters on
structured fields and user-defined parameters and results. Read the selected
run's terminal state, checks, measurements, logs, and relevant artifacts.
Compare revisions and inputs as well as outcomes; a rerun uses saved images,
not unbuilt notebook edits. Use leaf `--help` for filters and paging.

## Execution discipline

When executing, set an experiment budget and leave time to inspect and save results.
If a request times out, execution may still be running: inspect or stop the
owned operation before retrying. Do not replay cells with unknown effects.

Keep reusable logic in source modules and use cells for short probes and
calls. Confirm reusable code in a clean process before expanding to a suite.
Save remote files before releasing the owned session. Report separately what
was authored, what actually ran, what passed, and what remains unverified.
