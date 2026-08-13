# Manipulation physics — Isaac Sim 6.0.1 on Antioch

This is a reasoning guide for manipulation failures that look like control or
gripper bugs. It introduces the physics concepts and points to the detailed
Isaac references; it is not an API catalogue. Keep simulator imports inside
functions as required by the parent `isaac-sim-6` skill.

## Start with the collision shape

The visible mesh is not the contact shape. A convex hull replaces a flat jaw
pad with one enclosing hull, so a thin object can see a wedge instead of two
parallel faces. The resulting lateral force can cam the object away, leave a
gripper part-open, and make a placement failure look like bad IK. A convex
decomposition preserves useful local faces at a higher cost. Choose the
approximation from the contact geometry, then inspect the generated collision
shapes before tuning drives or friction.

Measure the contact-face offset at both ends of a jaw and compare it with the
object clearance. A millimetre-scale mesh error can dominate a centimetre-
scale grasp. The detailed rigid-body and approximation guidance is in
[`references/physics.md`](physics.md); grasp timing and the physical-versus-
welded fallback are in [`references/manipulation-grasping.md`](manipulation-grasping.md).

## Find colliders under instanced assets

USD instance proxies are visible in a composed stage only when traversal asks
for them. A converted robot can therefore contain many real colliders while a
plain `PrimRange` under `/World/Robot` reports none. `make_instanceable=False`
on an importer option does not prove that the authored subtree is editable.

Use the proxy traversal to inspect the composed stage, and make the owning
subtree non-instanceable before changing a collider or binding a material:

```python
def collider_prims(root):
    from pxr import Usd, UsdPhysics

    root.SetInstanceable(False)
    return [prim for prim in Usd.PrimRange(root, Usd.TraverseInstanceProxies(Usd.PrimAllPrimsPredicate)) if prim.HasAPI(UsdPhysics.CollisionAPI)]
```

An edit that binds a material to an empty result is a silent no-op. Assert that
the traversal found the expected collider family, bind a physics material to
the material prim, and then inspect the composed collision shape. Do not use a
successful return value from a binding helper as proof that a collider was
edited.

## Treat metrics as measurements, not verdicts

Physics offsets can create a constant-looking error. With `rest_offset` on both
bodies, a seat-distance metric can carry the sum of those offsets on every
episode. Establish a no-motion baseline, report the raw and baseline-corrected
values, and keep the offset policy with the run evidence. If every seed has the
same residual to the last decimal, inspect contact offsets and frame
conventions before blaming the controller.

An all-zero contact-force matrix can also be valid. A snapshot taken between
solver impulses, a contact below the reporting threshold, or a resting contact
with no net force can all produce zero. Pair force data with contact presence,
normals, impulses, separation, and relative motion. The matrix alone cannot
prove that a grasp is absent.

## Reset and timestep ordering for deformable chains

Reset is a physics state transition, not only a pose reset. In the pinned
image, `World.reset()` can clear PhysX GPU-dynamics enablement. Reapply the
setting and assert it after every reset, then start stepping; the complete
sequence is in [`references/physics-deformables.md`](physics-deformables.md).

Choose a timestep that resolves the fastest contact or chain motion. Stiff
contact chains normally need a smaller step and more substeps than a general
warehouse scene; the timestep and solver guidance is in
[`references/physics.md`](physics.md). Record the chosen timestep, substeps,
and reset ordering with the deformation and contact metrics so a replay can
separate numerical instability from a bad asset.

## Field register

The wave-9 findings registered as FB-150 (deformable reset and timestep),
FB-151 (valid zero contact force), FB-152 (instanced collider edits), and
FB-153 (collider approximation and metric bias). Use the detailed references
above for API changes and verify the exact pinned image before relying on a
newer Isaac release.
