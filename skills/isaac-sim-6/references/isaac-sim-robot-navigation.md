# Isaac Sim Robot Navigation — Runtime

Adapted from NVIDIA [`isaac-sim-robot-navigation/SKILL.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/isaac-sim-robot-navigation/SKILL.md) (Apache-2.0).

Read [Isaac Sim on Antioch](../SKILL.md) for startup, imports, and runtime configuration.
Native snippets run after startup, inside project functions or an active kernel.
Linked upstream scripts are source examples, not installed plugin commands.

## Purpose

Drive mobile robots at runtime with RL policies, trajectory followers, and physics/baked/per-frame stage strategies.

A rendered trajectory is not physical navigation evidence. Keep actuators, contacts, slip, balance, and obstacles active for a physical claim. Label kinematic replay and USD timeSamples as replay.

Dispatch and session setup belong to [antioch-platform](../../antioch-platform/SKILL.md). Verdicts and telemetry belong to [scenario-design](../../scenario-design/SKILL.md).

## Read these first

- [navigation-primitives.md](navigation-primitives.md) — occupancy maps, A* planning, differential/holonomic kinematics, robot footprints, look-at chase-camera math.
- [occupancy-map.md](occupancy-map.md) — ROS `map.yaml` / `map.png` generation.
- [rendering.md](isaac-sim-rendering.md) / [isaac-sim-rendering.md](isaac-sim-rendering.md) — capture and frame diagnosis.
- [isaac-sim-troubleshooting.md](isaac-sim-troubleshooting.md) — hang/freeze isolation.

## When to use this vs siblings

| Goal | Use |
|---|---|
| Drive a robot through a scene in real time | this reference |
| Record trajectories then re-render with sensors for SDG | [mobility-gen.md](mobility-gen.md) / [sdg.md](data-collection-sim.md) |
| Publish/subscribe Nav2 topics to ROS 2 | [antioch-platform](../../antioch-platform/SKILL.md); enable `isaacsim.ros2.bridge` through `SimulationConfig.extensions` |

## Runtime APIs

| Capability | API |
|---|---|
| Example RL policies | `isaacsim.robot.policy.examples.robots` (`SpotFlatTerrainPolicy`, `AnymalFlatTerrainPolicy`, `Go2FlatTerrainPolicy`, `H1FlatTerrainPolicy`, `FrankaOpenDrawerPolicy`) on `PolicyController` |
| Robot articulation | `isaacsim.core.experimental.prims.Articulation` |
| Physics lifecycle | `isaacsim.core.simulation_manager.SimulationManager` |

Enable `isaacsim.robot.policy.examples` in `SimulationConfig.extensions` before using those example classes; installed is not enabled. There is no `RobotPolicyRunner` / `get_*_spec` factory at this pin.

For shared primitives (omap, A*, kinematics, look-at), see [navigation-primitives.md](navigation-primitives.md).

## Physics stepping

Antioch defaults to PhysX. Select Newton only with `SimulationConfig(physics_engine="newton")` before startup and verify `SimulationManager.get_active_physics_engine()`.

Use one stepping owner: classic `World` / `SimulationContext`, experimental
`SimulationManager`, or a Lab environment loop. Use `antioch.world()` for the
classic singleton; do not create a second app or simulation context around
an existing owner.

Kit updates can advance a playing timeline but do not define a known physics timestep. With classic World, `World.step(render=True)` advances `render_dt / physics_dt` physics steps and `render=False` advances one. Count simulated time from `run.sim_s` or physics callbacks, not loop calls.

For experimental views, the timeline must be playing before tensor readback:

```python
import isaacsim.core.experimental.utils.app as app_utils

app_utils.play(commit=True)
```

Read physics-backed state (`Articulation.get_world_poses()`, DOF tensors). Those returns are batched even for one body. Isaac Sim quaternions are WXYZ.

### Heavy stages

Isolate expensive bodies, collision meshes, and rendering work in a small
reproduction. Keep support surfaces and obstacles needed by the task. Removing
all non-robot collision changes the experiment and cannot prove navigation.

## Viewport capture vs sensor cameras

Chase/overhead/POV look-at math lives in [navigation-primitives.md](navigation-primitives.md). Those helpers aim a viewport or authored `UsdGeom.Camera`; they do not replace `isaacsim.sensors.experimental.rtx` cameras or Replicator products. See [sensors.md](isaac-sim-sensor.md) and [isaac-camera.md](isaac-camera.md).

`omni.kit.viewport.utility.capture_viewport_to_file()` returns a `MultiAOVFileCapture`. Use `.wait_for_result(...)` rather than `.wait()`. A completion-frame count is a timeout, not proof of scene or renderer readiness; inspect the decoded frame.

## Failure modes

**`VkResult: ERROR_OUT_OF_DEVICE_MEMORY` during navigation render.** Baked USD timeSamples plus a heavy stage can exhaust GPU memory. Prefer per-frame transform updates, or reduce prim/render load. Do not kill workstation processes or relaunch Kit locally; the remote service owns the process.

**Robot floats above ground or falls through floor.** Wrong Z offset relative to collision geometry. Measure the footprint; see [navigation-primitives.md](navigation-primitives.md). Spawn at `z = ground + z_offset` from that measurement, not a hardcoded robot table.

**A\* path looks fine on the occupancy grid but the robot clips a rack in simulation.** Grid checks are approximate. Validate the swept footprint along the smoothed path, then use the physical controller for a physical claim. Teleporting to the goal is not controller evidence.

**Baked timeSamples / `bake_waypoints`.** Visualization or replay only. Do not convert a physics navigation task into keyframed animation and call it locomotion success.

## Integration

- Receives occupancy, A\*, kinematics, and camera math from [navigation-primitives.md](navigation-primitives.md).
- Receives `map.yaml` / runtime grids from [occupancy-map.md](occupancy-map.md).
- Receives robot USD from [urdf-mjcf-to-usd-conversion.md](urdf-mjcf-to-usd-conversion.md) / [usd-articulation.md](usd-articulation.md).
- Capture of nav runs: [rendering.md](isaac-sim-rendering.md).
