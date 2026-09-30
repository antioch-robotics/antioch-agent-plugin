---
name: agentic-simulation
version: "1.3.7"
description: Use when researching, developing, inspecting, testing, or improving simulations with Antioch, including controllers, custom kernels, solver integration, parameter fitting, datasets, policy experiments, and failure analysis. Choose a bounded research, Jupyter, script, or recorded evaluation loop and verify it with measurements and images.
---

# Agentic simulation

Turn a simulation question into grounded knowledge, working code, or a
measured result. Scale the work to the request: an answer needs no project,
and a small script need not become a suite. Use [Antioch platform](../antioch-platform/SKILL.md)
for project and compute setup.

## Choose an execution loop

| Work | Mechanism |
|---|---|
| Explore a scene, controller, or repeated trial | [Jupyter kernel](references/jupyter.md): keep Kit, scene, and variables alive between calls |
| Check a script from a clean process | `antioch service exec -- python src/main.py` |
| Save repeatable verdicts and compare cases | [Scenarios and suites](../scenario-design/SKILL.md) |

Keep reusable code in project modules. In Jupyter, edit locally, reload the
module, and call it again. Inspect state and images before changing the next
variable. A dispatched scenario starts a fresh process; try one case before
expanding to a large batch so warm kernel state cannot hide missing setup.
After a timeout, inspect the existing operation before repeating side effects.

## Connect the task to the right tools

Start from the question and the existing project. Use [research](../antioch-research/SKILL.md) to find a compatible API or method, [assets](../antioch-platform/references/assets.md) to reuse a robot or environment, and the [Isaac Sim](../isaac-sim-6/SKILL.md) or [Isaac Lab](../isaac-lab-3/SKILL.md) guide for the native implementation. Keep platform setup in [Antioch platform](../antioch-platform/SKILL.md).

For a custom kernel or solver integration, identify who owns each state array, its device and frame, and the order in which controls, forces, integration, and observations run. Test the coupled loop against a small known case. A Warp kernel compiling by itself does not prove it is called at the correct point in a Newton or Isaac step.

For parameter fitting, define the measured quantities and units, choose free parameters and plausible bounds, and save the baseline before searching. Hold out measurements for validation. Use [cases and suites](../scenario-design/SKILL.md#inputs-and-execution-policy) for independent comparisons and the [Python client](../antioch-platform/references/python-client.md) when an outer optimization loop needs to collect, submit, and read results.

For datasets, use [data collection](../isaac-sim-6/references/data-collection-sim.md) to define camera outputs, labels, variation, and writer behavior; inspect sample frames and annotations before scaling. For policies, use [Lab environment authoring](../isaac-lab-3/references/env-authoring.md) to establish observations, actions, resets, training, and evaluation. A saved run holds evidence, not a resumed simulator or training process.

## Measure the result

Decide what proves success: images for appearance, body state and contacts
for physical behavior, repeated cases for reliability. Read exceptions and
measurements and inspect actual images. An exit code of zero proves neither
physical correctness nor a passing evaluation.

Keep the requested physical model. Teleporting a robot, welding a payload,
or disabling collisions changes what the experiment proves. Test coupled
components together.

Retain useful code and evidence. Report what you wrote, what ran, what
passed, and what remains unverified.

## Further guidance

- For camera placement and inline images, load [navigation and capture](references/viewport.md).
- For datasets, load [data collection](../isaac-sim-6/references/data-collection-sim.md); for policies, load [Isaac Lab](../isaac-lab-3/SKILL.md).
- For a ROS 2 stack such as Nav2 or MoveIt 2 driving the simulated robot, load [ROS 2](../ros2/SKILL.md).
- For research, assets, environment changes, or saved runs, choose the relevant [platform guide](../antioch-platform/SKILL.md#capability-guides).

Bound experiments by the question: start with one seed, a short simulated interval, and the outputs needed to diagnose it. Expand cases or training only after that probe works and the user has authorized the larger run. Simulation evidence supports the modeled conditions; it does not replace validation on the physical system.
