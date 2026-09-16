# USD Articulation — Multi-Arm Robots

Adapted from NVIDIA [`usd-articulation/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-articulation/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Assemble and validate multi-link robot articulations with ArticulationRootAPI, fixed joints, and Isaac robot schema overlays before deployment.

A stage may contain several articulations; do not require one universal root or a fixed `ArticulationRootAPI` path for every scene. The "exactly one root" checklist below is for a **single multi-arm robot assembly**, not a facility stage.

Visual plausibility is not mechanical correctness. A vision-model PASS/FAIL on a render is not articulation evidence. After reset, check finite live poses, expected DOF names/counts, gravity response, a bounded command, and contact with the intended support.

Two layers stack on a robot USD:

1. **Physics**: `UsdPhysics.ArticulationRootAPI`, `UsdPhysics.Joint`, `UsdPhysics.RigidBodyAPI`, `PhysxSchema.PhysxArticulationAPI`.
2. **Robot Schema** (`usd.schema.isaac.robot_schema`): semantic overlay used by `isaacsim.robot.poser`, the importers, manipulators examples, and any tool that walks "this robot's links / joints / named poses".

Classic Franka examples live in `isaacsim.robot.manipulators.examples` (deprecated extension, not removed). Enable that extension ID through `SimulationConfig.extensions` when using those examples; the Python module path is not always the extension ID.

## Upstream helpers (source examples)

| Script | Purpose |
|---|---|
| [`scripts/apply_robot_schema.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-articulation/scripts/apply_robot_schema.py) | Apply Isaac Robot Schema overlay to an existing USD articulation |
| [`scripts/multi_arm_assembly.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-articulation/scripts/multi_arm_assembly.py) | Assemble a multi-arm robot in USD using FixedJoints |
| [`scripts/validate_articulation.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-articulation/scripts/validate_articulation.py) | Validate multi-arm robot USD articulation structure |

Not bundled commands.

## Multi-arm assembly (correct pattern)

`assemble_multi_arm_robot(stage, chassis_usd, arm_usd, robot_root, arm_offset, chassis_body_path, arm_base_path)` — spawn chassis + arm, create a FixedJoint, remove ArticulationRootAPI from the arm.

See [`scripts/multi_arm_assembly.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-articulation/scripts/multi_arm_assembly.py).

### What does not work

- Referencing the same arm USD twice via sublayer composition.
- Using `over "Geometry"` to add arms in a parts layer.
- Assuming visual presence equals physical attachment.

## Robot Schema overlay (Kit 110)

Apply the modern Isaac Robot Schema on top of the physics layer. The URDF/MJCF importers do this automatically; for hand-authored or retrofitted USDs apply it manually.

`apply_robot_schema(stage, robot_prim, link_prims, joint_prims, site_prims, robot_type)` — apply `IsaacRobotAPI` on root, `IsaacLinkAPI` on each link, `IsaacJointAPI` on each joint, `IsaacSiteAPI` on grasp/mount frames, then call `PopulateRobotSchemaFromArticulation` to fill `ROBOT_LINKS`/`ROBOT_JOINTS`. Valid `robot_type` tokens: Default, End Effector, Manipulator, Humanoid, Wheeled, Holonomic, Quadruped, Mobile Manipulators, Aerial.

See [`scripts/apply_robot_schema.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-articulation/scripts/apply_robot_schema.py).

| Schema | Applied to | Role |
|---|---|---|
| `IsaacRobotAPI` (`Classes.ROBOT_API`) | robot root | `robot_type`, ordered link/joint relations, named-pose container |
| `IsaacLinkAPI` (`Classes.LINK_API`) | each rigid link | mass/visual aux for tools that walk the chain |
| `IsaacJointAPI` (`Classes.JOINT_API`) | each joint | semantic joint metadata |
| `IsaacSiteAPI` (`Classes.SITE_API`) | grasp / mount / reference frames | named frames for IK targets, mounts, sensors |
| `IsaacNamedPose` (`Classes.NAMED_POSE`) | pose container prims | stored joint configurations (see [manipulation-ik.md](manipulation-ik.md)) |
| `Classes.SURFACE_GRIPPER` | end-effector site | author surface gripper via `CreateSurfaceGripper` |
| `Classes.ATTACHMENT_POINT_API` | site or link | attachment points used by accessory tooling |

## Validation checklist

Before trusting a **single multi-arm robot** (not a whole facility):

1. Exactly 1 `UsdPhysics.ArticulationRootAPI` on the chassis, nowhere else on that robot.
2. At least one chassis/root-branch `FixedJoint` anchors each top-level attached subsystem before flatten-before-deploy.
3. Every joint body is reachable from the single articulation root through one connected joint hierarchy.
4. `IsaacRobotAPI` present on the root; `robot_type` set to a valid token.
5. `ROBOT_LINKS` / `ROBOT_JOINTS` relations populated (call `PopulateRobotSchemaFromArticulation` if not).
6. Each grasp frame carries `IsaacSiteAPI` (not deprecated `IsaacReferencePointAPI` — deprecation is not removal).
7. Inspect composed bounds, joint graph, and physics APIs. Renders from several angles help authoring; they are not mechanical evidence.

Look for nested rigid bodies, missing or over-large colliders, filters, self-collision, mass/inertia, and initial penetration. Instance proxies are read-only: edit the source, make a deliberate override, or de-instance only the subtree that needs unique state.

## Validation snippet

`validate_articulation(usd_path)` — assert exactly 1 ArticulationRootAPI on that asset, verify hierarchy connectivity, and print root-attached children plus Robot Schema state.

See [`scripts/validate_articulation.py`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/usd-articulation/scripts/validate_articulation.py).

Extend with whatever subsystem path patterns your asset uses. Binary goal for this assembly: exactly one root, one connected articulated hierarchy, at least one root-branch `FixedJoint` anchor before flattening, and the Robot Schema overlay present.

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Arms render but float | No `FixedJoint` to chassis | add `FixedJoint` |
| Multiple articulation roots | Arm USD has its own root | `RemoveAPI(UsdPhysics.ArticulationRootAPI)` from arms |
| Training works but arms independent | Separate articulation trees | single root + `FixedJoint`s |
| `RobotPoser.solve_ik()` errors out | missing `IsaacRobotAPI` / link relations | `ApplyRobotAPI` + `PopulateRobotSchemaFromArticulation` |
| Importer applied `IsaacReferencePointAPI` | older asset | re-import with current Isaac Sim, or migrate to `IsaacSiteAPI` (deprecation warning) |
| `robot_type` attribute value rejected | typo or stale token | pick from `get_allowed_tokens(Attributes.ROBOT_TYPE)` |

Composition, layers, and delivery: [usd.md](usd-composition-architecture.md) / [usd-composition-architecture.md](usd-composition-architecture.md) / [usd-pipeline.md](usd-pipeline.md).
