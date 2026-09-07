# Manipulation and grasp validation

Use this for reaching, grasping, transport, placement, and choosing a motion
solver. The parent skill owns simulator startup; scenario-design owns durable
verdicts and evidence.

## Establish the robot contract

Inspect the composed robot before configuring IK (inverse kinematics).
Record the asset/version, physics backend, fixed/floating base, link and DOF
names, limits, units, and end-effector frame.

Measure a grasp frame from the intended contact geometry. It need not be the
gripper body's origin or the object's center. A schema site
(`usd.schema.isaac.robot_schema.ApplySiteAPI`) can make the frame reusable.
Do not copy a stock gripper offset into another asset.

An upstream robot example is a starting point, not a certificate for another
composition, payload, backend, or controller. Private run IDs and historical
tuning tables do not establish a reusable platform capability.

## Pick a motion stack

| Surface | Suitable work | Verify before use |
|---|---|---|
| `isaacsim.core.experimental.prims.Articulation` | Joint control and Jacobian-based differential IK | Live array shapes, link row, DOF columns, frames, limits |
| `isaacsim.robot.poser` | Schema-based offline IK and named-pose authoring | Robot schema, solver result and pose storage signatures |
| `isaacsim.robot_motion.pink` | Pinocchio/PINK IK | Robot model, task costs, integration period and limit handling |
| `isaacsim.robot_motion.cumotion` | Motion generation with a world model | Supported robot config, obstacle updates, state types and reset lifecycle |
| `isaacsim.robot_motion.motion_generation` | Existing Lula/RMPflow integrations | Retained API at the selected pin; do not confuse it with the `lula` extension module |

Choose the smallest solver that meets the task. Do not migrate a working
controller solely because another stack is newer. Retrieve an example from
the pinned source for the chosen robot/backend and read its initialization
and per-step calls together.

Named poses suit known, unchanged scenes. Reusing a pose without checking
current object and obstacle state is not reactive control.

## Jacobians and array shape

Experimental articulation APIs are batched, even for one robot. Preserve the
batch axis or select the intended robot explicitly; a position array shaped
`(1, 3)` is not indexed as `position[2]`.

For a single articulation, the pinned PhysX Jacobian layout is:

```text
fixed base:    (num_links - 1, 6, num_dofs)
floating base: (num_links,     6, num_dofs + 6)
```

The returned collection also has an articulation batch axis. Fixed-base
Jacobians omit the root link's row. Do not apply that row offset to a floating
base, assume free-root columns are at the end, or assume finger joints are
the trailing columns. Resolve indices from the actual articulation and the
backend's documented layout.

For an IK step, check the pose-error frame, quaternion order, selected
columns, joint limits, and maximum commanded motion. A finite solver result
is not enough: verify forward kinematics and the resulting live motion.
Do not slice an unexpected tensor until its dimensions happen to fit.

Under-actuated arms cannot satisfy arbitrary six-dimensional targets. Choose
reachable constraints and evaluate residuals and limits. Joint interpolation
can be useful, but fewer than six DOFs does not make it a universal fix.

## Grasp mechanics

Use the mechanism that matches the requested task:

- Finger contact evaluates collision geometry, friction, effort limits, and
  payload dynamics.
- `isaacsim.robot.surface_gripper` models attachment-style grippers. Closing
  is a command, not an immediate proof of attachment. Step and observe status
  and physical payload motion with a bounded deadline.
- A `UsdPhysics.FixedJoint` is an explicit constraint. It can model a
  deliberate attachment, but cannot prove that friction alone holds a grasp.

Never silently weld, teleport, or make a payload kinematic to make a physical
grasp pass. If the user authorizes a simplified model, label it and state what
the test no longer proves.

For a deliberate fixed-joint attachment, derive both joint frames from live
body poses at attachment time. Account for batching and quaternion ordering.
A hard-coded inconsistent frame can snap the bodies together. On release,
remove only the constraint created by that operation.

Inspect collision approximation before increasing drive effort: a convex
hull can fill a jaw opening. Resolve instance-proxy edits deliberately.
Tune gains and timestep for the actual masses and actuator model; there is no
robot-independent table of safe stiffness or grasp force.

## Acceptance evidence

Choose thresholds from the task before running. Observe at least:

1. **Engagement:** contact or attachment state and the measured object-to-tool
   transform. Proximity alone does not prove a grasp.
2. **Lift and transport:** sustained height above the support surface, payload
   motion with the tool, relative-transform drift, and unintended collisions.
   A single peak height or final sample can miss a drop.
3. **Release and placement:** the object is detached, lies within the target
   volume (including height), and remains settled for a measured simulated
   interval. Low speed and XY error alone can pass an object on the wrong
   shelf or floor.

Compare relative-transform change for slip; absolute distance to a tool origin
can include an intentional offset. Accumulate worst-case failures through the
phase so a later good sample cannot overwrite an earlier failure.
Use live physics state, not an unsynchronized USD transform cache.

Save measured values with `run.add_result` and named verdicts with `run.check`.
Retain diagnostic frames on failure. Smooth motion or a final image is not
evidence that all manipulation phases passed.
