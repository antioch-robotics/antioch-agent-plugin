# Grasping and pick-and-place validation — Isaac Sim 6.0.1

Detail reference for the manipulation domain of the `isaac-sim-6` skill.
Everything here runs on Antioch machines. Use the stream to debug motion, then
validate numerically and save every checkpoint with `run.add_result` so the
evidence survives the run.

## FixedJoint grasping (rigid objects)

The one rule that prevents PhysX explosions: **compute the gripper→object
relative transform at the moment of contact** from simulated poses — never
hardcode it. A hardcoded offset plus a stiff drive snaps the pair to an
inconsistent configuration and the sim diverges.

```python
def grasp_with_fixed_joint(stage, obj, gripper_path: str, obj_path: str) -> str:
    from isaacsim.core.experimental.prims import RigidPrim
    from pxr import Gf, UsdPhysics

    grip_pos, grip_quat = RigidPrim(gripper_path).get_world_poses()
    obj_pos, obj_quat = obj.get_world_poses()
    rel_pos, rel_quat = relative_pose(grip_pos.numpy(), grip_quat.numpy(), obj_pos.numpy(), obj_quat.numpy())
    joint_path = f"{gripper_path}/GraspFixedJoint"
    joint = UsdPhysics.FixedJoint.Define(stage, joint_path)
    joint.CreateBody0Rel().SetTargets([gripper_path])
    joint.CreateBody1Rel().SetTargets([obj_path])
    joint.CreateLocalPos0Attr().Set(Gf.Vec3f(*rel_pos))
    joint.CreateLocalRot0Attr().Set(Gf.Quatf(*rel_quat))
    return joint_path
```

`relative_pose` is a small WXYZ-quaternion helper — implement it with scipy at
the boundary (see the quaternion note in `isaac-sim-6`), converting
`[w, x, y, z]` → `[x, y, z, w]` and back.

Release by deleting the joint prim, then step ≥ 1 s of physics before judging
the place:

```python
def release_fixed_joint(stage, joint_path: str) -> None:
    stage.RemovePrim(joint_path)
```

## SurfaceGripper (vacuum / magnetic)

Authored on the robot with `usd.schema.isaac.robot_schema.CreateSurfaceGripper`
(typically during asset prep — see `references/assets.md`). At runtime:

```python
def close_gripper(end_effector_path: str) -> None:
    from isaacsim.robot.surface_gripper import _surface_gripper

    iface = _surface_gripper.acquire_surface_gripper_interface()
    gripper_path = f"{end_effector_path}/SurfaceGripper"
    iface.close_gripper(gripper_path)
    status = iface.get_gripper_status(gripper_path)  # GripperStatus.{Open,Closing,Closed}
    assert status == _surface_gripper.Closed, f"gripper failed to engage: {status}"
```

`iface.open_gripper(gripper_path)` releases. The status passes through the
transient `Closing` member first, so assert on settled frames, not the same
step as `close_gripper`. Treat the status check as a checkpoint of its own —
"commanded close" is not "grasped".

**`Closing` can persist indefinitely with a real grasp** (observed live: a
gripper held nonzero contact force through an entire pick while the status
never reached `Closed`). Do not gate success on `Closed` alone — verify the
grasp physically instead: nonzero contact-sensor force at the pad, or the
payload tracking the end effector across steps. Where the status wedges but
force is real, the working fallback is to weld the payload with a
`UsdPhysics.FixedJoint` created at the contact frame on grasp and removed on
release.

## When you take the kinematic fallback

Substituting a weld or kinematic attachment for a physical grasp is a
reporting decision, not just a code change:

- Record the failed physical probe first — the gripper status timeline and the
  contact evidence go into the run (metrics/artifacts) before you switch.
- Keep the placement motion as real robot motion; the fallback replaces the
  grasp mechanics, never the trajectory.
- Report settle results as kinematic-settle, not dynamic proof: a welded
  payload cannot slip, so lift-slip and rest-speed no longer prove what they
  prove for a physical grasp.
- Never loosen the gate to hide the substitution — thresholds stay, and the
  saved run results state the fallback.

## Drive-gain tuning for grasped transport

| Parameter | Conservative | Moderate | Aggressive |
|---|---|---|---|
| Arm drive `stiffness` | 200 | 400 | 800 |
| Arm drive `damping` | 40 | 60 | 80 |
| IK `damping` (DLS λ) | 0.1 | 0.05 | 0.01 |
| Max joint delta per step | 0.02 rad | 0.05 rad | 0.10 rad |

Payload changes the dynamics: re-run the conservative row with the grasp
attached before moving up. Finger drives for friction grasps need enough
stiffness to hold against gravity — verify via the lift-slip metric below, not
by feel.

## The three-checkpoint validation loop

Run this as part of the scenario, after each phase. Each checkpoint asserts on
simulated state and emits a metric; a failed assertion fails the run with the
metric values still recorded. Thresholds: grasp 2 cm, lift-slip 2 cm, place
5 cm, residual speed 1 cm/s — loosen only as a deliberate, stated decision.

```python
def check_grasp(run, obj, site, threshold_m: float = 0.02) -> None:
    """Checkpoint 1: object center sits at the grasp site."""
    import numpy as np

    obj_pos, _ = obj.get_world_poses()
    site_pos, _ = site.get_world_poses()
    offset = float(np.linalg.norm(obj_pos.numpy() - site_pos.numpy()))
    run.add_result("grasp_offset_m", offset)
    assert offset < threshold_m, f"grasp offset {offset:.3f} m exceeds {threshold_m} m"


def check_lift(run, obj, site, obj_z_before: float, lift_m: float) -> None:
    """Checkpoint 2: object rises with the gripper and does not slip."""
    import numpy as np

    obj_pos, _ = obj.get_world_poses()
    site_pos, _ = site.get_world_poses()
    delta_z = float(obj_pos.numpy()[2] - obj_z_before)
    slip = float(np.linalg.norm(obj_pos.numpy() - site_pos.numpy()))
    run.add_result("lift_delta_z_m", delta_z)
    run.add_result("lift_slip_m", slip)
    assert delta_z >= 0.8 * lift_m, f"lifted only {delta_z:.3f} m of {lift_m} m"
    assert slip < 0.02, f"object slipped {slip:.3f} m from grasp site during lift"


def check_place(run, obj, target_xyz, settle_steps: int = 120) -> None:
    """Checkpoint 3: object rests on target after release + settle."""
    import numpy as np
    import isaacsim.core.experimental.utils.app as app_utils

    for _ in range(settle_steps):  # >= 1 s at 120 Hz
        app_utils.update_app()
    obj_pos, _ = obj.get_world_poses()
    lin_vel, _ = obj.get_velocities()
    error = float(np.linalg.norm(obj_pos.numpy()[:2] - np.asarray(target_xyz[:2])))
    speed = float(np.linalg.norm(lin_vel.numpy()))
    run.add_result("place_error_m", error)
    run.add_result("residual_speed_mps", speed)
    assert speed < 0.01, f"object still moving at {speed:.3f} m/s"
    assert error < 0.05, f"place error {error:.3f} m exceeds 0.05 m"
```

Notes on reading this correctly:

- `obj_z_before` must be captured from `get_world_poses()` *before* the lift
  command — a captured authored pose (XformCache) poisons the lift check.
- The lift-slip metric catches the classic failure where the FixedJoint exists
  but the offset is wrong: the object lifts, just not with the gripper.
- Place is judged on XY distance plus rest speed; Z is implied by "resting on
  the target surface" once the object is static.
- For friction grasps, add a contact-force assertion at checkpoint 1 (physics
  sensors are covered in `isaac-sim-6` `references/physics.md`): a minimum
  normal force on each finger proves engagement.

A run is a successful pick-and-place only when all three checkpoints pass with
their metrics in the record. Motion that looks smooth in telemetry but fails a
checkpoint is a failed run — report it as such.
