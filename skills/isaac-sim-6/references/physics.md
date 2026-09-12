# Physics, backends, and live state

Use this for scene setup, collisions, drives, contacts, deformables, and
physics failures. The parent skill owns startup and the engine pin.

## One backend and one stepping owner

Antioch defaults to PhysX. Select Newton with
`SimulationConfig(physics_engine="newton")` before startup, or on the scenario
decorator. Check `SimulationManager.get_active_physics_engine()` when backend
identity matters.

The pinned Newton extension enables `auto_switch_on_startup` by default.
Do not assume enabling it merely registers an inactive engine. Antioch's
startup config is the supported way to choose the backend before the stage
opens; a conflicting second startup cannot change it.

PhysX and Newton do not have identical features or tensor behavior. The
standalone `newton` solver library and Isaac's Newton USD/tensor integration
are different surfaces. A feature in `newton.ModelBuilder` does not establish
support for the same feature through an Isaac scene. Retrieve the chosen
backend's implementation and test that exact path.

Use the stepping owner already selected by the application: classic
`World`/`SimulationContext`, the experimental `SimulationManager` workflow, or
Isaac Lab's environment loop. Do not mix several independent stepping loops.
Kit app updates can advance a playing timeline; they are not a substitute for
a known simulation timestep and render cadence.

## Scene and body setup

Confirm that the stage has an appropriate physics scene before starting.
`/World/PhysicsScene` is a convention, not a required path. A framework may
create its own scene; do not add a conflicting second one.

Read stage units and up-axis before setting gravity or geometry. For a simple
rigid body, apply `UsdPhysics.RigidBodyAPI` and collision to the body geometry.
For a compound body, child colliders belong to their nearest rigid-body
ancestor. An unintended nested rigid body changes that ownership.

Check these independently:

- Visual geometry versus collision geometry and approximation.
- Static, dynamic, and kinematic behavior.
- Mass, center of mass, and inertia in the stage's units.
- Initial penetrations, collision filters, and self-collision.
- Material binding, friction, restitution, and contact offsets.
- Joint frames, limits, actuator mode, stiffness, damping, and effort limits.

A convex hull can close a gap that is visible in the mesh. Nonuniform scale
can change collision behavior; inspect the cooked representation and contacts.
Increasing solver iterations cannot repair a wrong collision shape.

USD `physics:angularVelocity` is in degrees per second. Do not carry that
unit into a tensor API without checking its contract; runtime APIs commonly
use radians per second. Quaternions and batched shapes have similar boundary
risks.

## Initialization and readback

Author initial conditions before reset. Initialize runtime views through the
selected framework, then apply runtime targets or velocities through its live
API. Do not confuse a drive target with a direct state write.

During simulation, read physics-backed state such as
`RigidPrim.get_world_poses()` and `Articulation` state. These APIs are batched;
one body's position normally still has shape `(1, 3)`.

USD transform reads reflect the composed USD state. Whether physics writes
current transforms back into that state depends on backend and settings.
An old `UsdGeom.XformCache` also retains cached transforms until cleared.
Thus a USD read can be stale, but it is not universally "the initial pose
forever." Enabling an arbitrary writeback setting does not prove that every
backend and renderer now agree. Check the selected source of truth and frame.

## Stability and contact tests

Choose timestep and solver settings for the task's speeds, contact geometry,
masses, and accuracy. Use a convergence comparison when changing numerical
settings: smaller timesteps or different solver budgets should not change the
claimed result beyond its tolerance. Know what one step call advances: the
SDK requires `render_dt` to be an integer multiple of `physics_dt`, and with
`physics_dt` the smaller of the two, `World.step(render=True)` advances
`render_dt / physics_dt` physics steps while `World.step(render=False)`
advances exactly one, so a loop that renders every N-th call advances more
physics than N steps per N calls. Take simulated time from the engine —
`run.sim_s`, or the physics dt times the physics callback count — never from
an authored call counter.

A universal three-second settle time, fixed iteration count, or maximum
number of bodies is not a physical contract. Measure residual motion over a
declared simulated interval and enforce a bounded wall-clock timeout.

For contacts, verify sensor filters, reporting thresholds, force/impulse units,
and sampling time. A contact reading taken over a render step is an impulse
summed over `render_dt / physics_dt` substeps: divide it by the interval it
was accumulated over, not by `physics_dt`. No report can mean a filter or
reporting problem rather than no contact. For momentum or energy tests, account
for gravity, damping, friction, actuators, and solver error before declaring
an engine limitation.

Do not replace contacts with analytical motion to make a physics test pass.
A simplified model requires an explicit change in the requested evidence.

## Deformables and Newton-native work

First distinguish the path:

- PhysX/USD deformables use their schema, cooked mesh, material, and attachment
  requirements.
- Isaac's Newton integration uses its supported USD and tensor features.
- Standalone Newton/Warp code builds and steps its own model and solver.

Retrieve the exact solver example at the runtime pin. Keep array device,
dtype, quaternion convention, ownership, and synchronization explicit. Do not
apply a PhysX-only schema and assume Newton honors it.

For reset failures, reproduce with one object and inspect topology, allocation,
and state-reset order. Distinguish a Python API failure, a CUDA kernel failure,
and a process crash. Preserve the first error, backend/version, and minimal
input; do not keep retrying a faulted GPU process as if its state were valid.

Validate deformation, contacts/attachments, finite state, and repeatable
reset in the selected runtime. A successfully imported class proves none of
these behaviors.
