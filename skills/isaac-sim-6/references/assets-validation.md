# Articulation validation

Depth reference for the assets domain of the `isaac-sim-6` skill. Run this before trusting
any imported, retrofitted, or hand-assembled robot. Visual plausibility is
not mechanical correctness — validate numerically, not only through the stream.

## Checklist

1. Exactly one `UsdPhysics.ArticulationRootAPI`, on the chassis — nowhere
   else.
2. Every limb connects to the chassis via a `FixedJoint` (rigid mount) or
   an articulated joint chain — no floating geometry.
3. Joint count matches the design: N arm joints per arm + chassis joints.
4. `IsaacRobotAPI` present on the root; `robot_type` is a valid token.
5. `ROBOT_LINKS` / `ROBOT_JOINTS` relations populated — run
   `PopulateRobotSchemaFromArticulation` if empty.
6. Grasp / mount frames carry `IsaacSiteAPI`, not the deprecated
   `IsaacReferencePointAPI`.
7. If the run renders captures, read the artifact back after download and
   confirm attachment visually — never eyeball a live GUI (there is none).

## Validation script

Self-contained; reads USD only, so it needs no simulator boot — but it must
run on the machine (that is where `pxr` and the robot schema live). Save as
`validate_articulation.py` and dispatch with
`antioch run validate_articulation.py -- /path/to/robot.usd` — the report
prints straight to your terminal. A `run` script is not a scenario run, so
there is no run id for `antioch scenario show SCENARIO_RUN_ID --logs` or
`antioch scenario download SCENARIO_RUN_ID`; for a downloadable report, run
the same checks inside a scenario and keep the output with
`run.add_artifact(...)`.

```python
"""Validate an articulation USD: one root, joint connectivity, robot schema."""

import sys


def validate(usd_path: str) -> None:
    from pxr import Usd, UsdPhysics
    from usd.schema.isaac.robot_schema import Attributes, Classes, Relations
    from usd.schema.isaac.robot_schema.utils import GetAllNamedPoses

    stage = Usd.Stage.Open(usd_path)

    # 1. Exactly one articulation root
    roots = [p for p in stage.Traverse() if p.HasAPI(UsdPhysics.ArticulationRootAPI)]
    root_paths = [str(p.GetPath()) for p in roots]
    print(f"ArticulationRootAPI count: {len(roots)} -- {root_paths}")
    assert len(roots) == 1, f"Expected exactly 1 articulation root, found {len(roots)}: {root_paths}"
    chassis_path = root_paths[0]

    # 2/3. Joints and their connected bodies; FixedJoint attachment to the chassis
    joints = [p for p in stage.Traverse() if p.IsA(UsdPhysics.Joint)]
    attached = set()
    for joint_prim in joints:
        joint = UsdPhysics.Joint(joint_prim)
        body0 = [str(t) for t in joint.GetBody0Rel().GetTargets()]
        body1 = [str(t) for t in joint.GetBody1Rel().GetTargets()]
        print(f"  {joint_prim.GetPath()} ({joint_prim.GetTypeName()}): {body0} -> {body1}")
        if joint_prim.IsA(UsdPhysics.FixedJoint) and any(t.startswith(chassis_path) for t in body0):
            attached.update(body1)
    print(f"Joints: {len(joints)} | FixedJoint-attached children: {sorted(attached)}")

    # 4/5/6. Robot Schema overlay
    robots = [p for p in stage.Traverse() if p.HasAPI(Classes.ROBOT_API.value)]
    print(f"IsaacRobotAPI count: {len(robots)}")
    for robot in robots:
        robot_type = robot.GetAttribute(Attributes.ROBOT_TYPE.name).Get()
        links = robot.GetRelationship(Relations.ROBOT_LINKS.name).GetTargets()
        joints_rel = robot.GetRelationship(Relations.ROBOT_JOINTS.name).GetTargets()
        poses = GetAllNamedPoses(stage, robot)
        print(f"  {robot.GetPath()}: robot_type={robot_type!r} links={len(links)} joints={len(joints_rel)} named_poses={list(poses)}")

    deprecated = [p for p in stage.Traverse() if "IsaacReferencePointAPI" in p.GetAppliedSchemas()]
    if deprecated:
        print(f"WARNING: deprecated IsaacReferencePointAPI on {[str(p.GetPath()) for p in deprecated]} — migrate to IsaacSiteAPI")


if __name__ == "__main__":
    validate(sys.argv[1])
```

Extend the path patterns to your asset's naming (`BaseMount`, `wrist`,
etc.). The binary goal: exactly one root, every limb reachable from the
chassis through joints, and the Robot Schema overlay present and populated.

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Arms render but float | no `FixedJoint` to chassis | add the joint |
| Multiple articulation roots | arm USD has its own root | `RemoveAPI(UsdPhysics.ArticulationRootAPI)` on arms |
| Training runs but arms move independently | separate articulation trees | single root + `FixedJoint`s |
| Poser / IK tools error out | missing `IsaacRobotAPI` or relations | `ApplyRobotAPI` + `PopulateRobotSchemaFromArticulation` |
| Importer wrote `IsaacReferencePointAPI` | older asset | re-import with 6.0.1, or migrate to `IsaacSiteAPI` |
| `robot_type` rejected | typo or stale token | `GetAttribute(Attributes.ROBOT_TYPE.name).GetMetadata("allowedTokens")` |
