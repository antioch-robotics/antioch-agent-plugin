# Navigation, maps, and mobile robots

Use this for occupancy maps, route planning, wheel control, and legged policy
integration. Match the method to the requested evidence: a rendered trajectory
is not proof that a robot can drive it.

## Choose the execution model

- Physical navigation evaluates actuators, contacts, slip, balance, collision,
  and the controller. Keep physics and obstacles active.
- Kinematic replay is useful for visualization or replaying recorded sensor
  poses. Label it as replay; teleporting a robot does not validate control.
- A large stage is a reason to profile loading, collision cooking, and memory,
  not to strip obstacle physics or silently switch the test to replay.

For differential and holonomic control, start from
`isaacsim.robot.experimental.wheeled_robots`. Resolve wheel names and geometry
from the actual asset. Check the controller's input units and output order;
apply velocity targets to the matching joints, not guessed indices.
A legged policy also depends on observation order, normalization, action
scale, joint order, control period, and the robot version it was trained on.

## Build the map in a declared frame

Choose resolution, origin, bounds, vertical obstacle band, and unknown-space
policy before generating a grid. Include collision geometry that can contact
the robot over that height band; exclude a traversable floor without excluding
low obstacles or overhangs that hit the body.

The pinned occupancy generator is
`isaacsim.asset.gen.omap.bindings._omap.Generator`. Retrieve its initialization,
timeline/physics prerequisites, occupancy values, and buffer layout. Do not
infer array orientation from a flattened buffer.

A projection of USD bounding boxes is only an approximation. It can overfill
hollow geometry, miss instance descendants, or mark an entire floor occupied.
Use it only when that approximation fits the task, and inspect an asymmetric
fixture with a known obstacle in one corner.

## Footprint and clearance

A world-aligned bounding box changes as the robot rotates. Do not rotate that
box a second time as if it were the robot's local footprint. Measure the
collision footprint in the robot frame and track the reference point used by
the map and controller.

- The **circumscribed** radius encloses the entire footprint and gives a
  conservative circular approximation.
- The **inscribed** radius fits inside it. For a rectangular robot, it omits
  the corners and cannot certify clearance at every yaw.
- Convert a required nonnegative clearance to grid cells with
  `ceil(clearance_m / resolution_m)`. Rounding or flooring can under-inflate.
- Reject any candidate footprint outside the map. Polygon rasterizers often
  clip silently, which can turn an out-of-bounds pose into a false clear cell.
- Treat unknown space according to the task's explicit policy, normally as
  blocked when proving a collision-free route.

A path must clear the swept footprint along its segments and rotations, not
just at waypoints. Smoothing can introduce collisions; validate the smoothed
path again. Grid validation is still an approximation, so confirm with the
physical controller when physical navigation is the requested outcome.

For PhysX scene queries, check filter and callback semantics in the pinned
API. Exclude the queried robot and explicitly classify the support surface.
An "any overlap" callback can otherwise report the robot itself or its floor.

## ROS map export

ROS map metadata and image orientation must describe the same world frame.
For an internal Cartesian grid where row zero is minimum world Y, flip rows
once when writing an image whose top row is maximum world Y. Do not flip twice
on load. Verify this with asymmetric obstacles and known world coordinates.

For conventional trinary maps, encode free, occupied, and unknown distinctly
and set matching `negate`, `occupied_thresh`, and `free_thresh` metadata.
Do not map every nonzero value to occupied if one of those values means
unknown. Preserve resolution and origin, including the origin's yaw.

Load the exported image/YAML through the actual downstream map reader and
check several world-to-cell samples before driving. Merely counting occupied
pixels cannot detect a vertical mirror or a wrong origin.

## Route evidence

Keep the planned route, actual base poses, minimum clearance, collision
events, progress, and final goal error. For legged robots include falls and
stability; for wheeled robots include stalls and slip where relevant.
Use task-defined tolerances and a bounded timeout. A rendered route through
walls or an endpoint reached by teleportation is a failed physical test.

ROS bridge setup belongs to the platform skill's ROS 2 reference.
MobilityGen record/replay belongs to `references/sdg.md`.
