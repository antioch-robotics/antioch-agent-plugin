# Controllers, RL Policies, and Execution Models

Driving the robot and choosing how the run executes. All simulator imports
stay inside function bodies (Rule #1, `isaac-sim-6`); velocity targets are
set **after** `play(commit=True)` (boot ordering, `isaac-sim-6`).

## Differential drive — `DifferentialController` + experimental `Articulation`

Kit 110 pattern: the controller converts a body twist `[linear, angular]`
into left/right wheel velocities; you write them as DOF velocity targets.

```python
def drive_step(
    robot: "Articulation", dc: "DifferentialController", command: "np.ndarray", left_wheel_indices: list, right_wheel_indices: list, num_dofs: int
) -> None:
    import numpy as np

    wheel_vels = dc.forward(command)  # command = [linear_speed, angular_speed]
    targets = np.zeros((1, num_dofs))
    for i in left_wheel_indices:
        targets[0, i] = wheel_vels[0]
    for i in right_wheel_indices:
        targets[0, i] = wheel_vels[1]
    robot.set_dof_velocity_targets(targets)


def make_robot(prim_path: str, wheel_radius: float, wheel_base: float) -> tuple:
    from isaacsim.core.experimental.prims import Articulation
    from isaacsim.robot.experimental.wheeled_robots.controllers import DifferentialController

    robot = Articulation(prim_path)
    dc = DifferentialController(wheel_radius=wheel_radius, wheel_base=wheel_base)
    return robot, dc
```

The raw kinematics, for when you hand-roll the conversion:

```python
def body_to_wheel(vx: float, omega: float, track_w: float, wheel_r: float) -> tuple:
    v_left = (vx - omega * track_w / 2) / wheel_r
    v_right = (vx + omega * track_w / 2) / wheel_r
    return v_left, v_right
```

### Waypoint steering parameters (starting points, tune per robot)

- PD heading control: KP ≈ 2.0–2.5, KD ≈ 0.5–1.2, max angular ≈ 1.0–1.5 rad/s.
- Slow near waypoints and under large heading error; waypoint tolerance
  ~4 m for yard-scale routes, tighter indoors.
- Speed schedule: ~0.8 m/s in cluttered aisles, ~1.5 m/s in open corridors.
- For fast wheels on stiff contact, step physics fine: dt = 1/120 s with
  substeps (physics configuration is owned by `isaac-sim-6`
  `references/physics.md`).
- Watchdog: mark the run failed if |z| or |x|/|y| leaves the site bounds —
  a runaway robot otherwise burns machine time until the timeout.

## Holonomic / mecanum

Use `isaacsim.robot.wheeled_robots.controllers.holonomic_controller.HolonomicController`.
Extract wheel positions, orientations, and mecanum angles from the robot
USD rather than hardcoding them (`HolonomicRobotUsdSetup` in
`isaacsim.robot.wheeled_robots.robots`), and feed a 3D command `[forward, lateral, yaw]`. A 2D
`[linear, angular]` navigation command maps to `[linear, 0, angular]`.

## RL policy loading — `PolicyController`

Load a trained policy checkpoint (e.g. an rsl_rl export) at runtime and
step it each physics tick. This is the runtime-execution path — training
itself is out of scope (the SDG reference, `references/sdg.md`, and the Isaac
Lab skills cover adjacent ground).

`PolicyController` is an abstract base — a usable controller subclasses it
and implements `_compute_observation` (the observation tensor in the
structure env.yaml specifies) and `forward`. The concrete example
controllers in `isaacsim.robot.policy.examples.robots`
(`Go2FlatTerrainPolicy`, `SpotFlatTerrainPolicy`, `H1FlatTerrainPolicy`, …)
are working references:

```python
def make_policy(robot_prim: str, policy_path: str, params_path: str) -> "PolicyController":
    from isaacsim.robot.policy.examples.robots import Go2FlatTerrainPolicy

    return Go2FlatTerrainPolicy(
        prim_path=robot_prim,
        policy_path=policy_path,  # policy checkpoint (.pt / .onnx)
        env_config_path=params_path,  # env.yaml with the observation structure
    )
```

Usage per tick after `play(commit=True)`: call `controller.forward()` — it
computes the observation, runs the policy, and applies the action to the
robot's DOF targets internally, returning None (subclasses may extend the
signature, e.g. Go2's `forward(dt, command)`). Policy checkpoints are data
files — publish them as scenario assets rather than baking paths into code
(`antioch assets`, `antioch-platform`).

## Execution models for heavy stages

### Model A — physics

Timeline playing; the controller drives joints every step. Correct when
dynamics matter (contact, slip, policy rollouts) and the stage is light.

### Model B — baked timeSamples

Pre-compute the trajectory, write xform timeSamples on the robot root,
scrub the timeline. Cheapest per frame, but the baked animation plus RT
rendering on a 50K+ prim stage OOMs the GPU (see below). Small stages only.

### Model C — per-frame transform (heavy-stage default)

Set the robot pose directly each frame and step once:

```python
def follow_trajectory(robot: "Articulation", waypoints: "np.ndarray") -> None:
    import numpy as np
    import isaacsim.core.experimental.utils.app as app_utils

    for x, y, yaw in waypoints:
        quat = yaw_to_quat_wxyz(yaw)  # [w, x, y, z]
        robot.set_world_poses(positions=np.array([[x, y, 0.0]]), orientations=np.array([quat]))
        app_utils.update_app()
```

No dynamics, no OOM, deterministic — the right choice for validating routes
and producing navigation artifacts on warehouse-scale stages. The pose you
set is a teleport: do not use this model when contact behavior matters.

## GPU OOM avoidance

- **Symptom**: `VkResult: ERROR_OUT_OF_DEVICE_MEMORY` mid-run on a heavy
  stage → **cause**: baked timeSamples + RT rendering + 50K+ prims →
  **fix**: switch to per-frame transforms (Model C).
- **Symptom**: sim hangs or crawls on a 30K+ prim stage with many physics
  bodies → **fix**: strip physics from every non-robot prim before playing:

```python
def strip_non_robot_physics(robot_prefix: str) -> None:
    from pxr import Usd, UsdPhysics

    import antioch

    stage = antioch.stage()
    for prim in Usd.PrimRange(stage.GetPseudoRoot()):
        if str(prim.GetPath()).startswith(robot_prefix):
            continue
        if prim.HasAPI(UsdPhysics.RigidBodyAPI):
            prim.RemoveAPI(UsdPhysics.RigidBodyAPI)
        if prim.HasAPI(UsdPhysics.CollisionAPI):
            prim.RemoveAPI(UsdPhysics.CollisionAPI)
```

Note the trade: stripped prims no longer collide, so the robot can pass
through shelves. Strip only when Model C (kinematic playback) makes
collisions irrelevant, or keep collision but remove `RigidBodyAPI` so
statics stay solid without solver cost.
- A crashed or OOMed run leaves no dirty local state — each Antioch
  execution is one fresh process per slot. Fix and re-dispatch; never
  design around "cleaning up the GPU" locally.

## Verify the run

Use the browser stream to debug the route, then prove the navigation
numerically and save it as run output:

- Waypoint-reaching rate, path length vs. planned length, min clearance to
  obstacles along the executed trajectory.
- Final pose vs. goal pose (position and yaw error).
- The occupancy grid, planned path, and executed trajectory rendered into a
  PNG artifact; `.rrd` telemetry for time-series (pose, wheel commands).
