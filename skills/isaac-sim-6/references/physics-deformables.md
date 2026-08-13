# Deformables — Isaac Sim 6.0.1 on Antioch

This reference separates two surfaces that are easy to conflate:

1. the Isaac Sim / PhysX USD-schema integration layer (`DeformableBodyAPI`,
   experimental deformable prims, and PhysX GPU settings); and
2. the top-level `newton` runtime package, whose `ModelBuilder` can build and
   solve soft grids with VBD.

All simulator imports below stay inside functions. The facts labelled
**live-verified** come from the pinned softpack run; names and signatures come
from the harvested Isaac 6.0.1/Newton stubs.

## Import reality

`isaacsim.physics.newton` appears in the extension/stub namespace, but importing
that module in the pinned image raised `ModuleNotFoundError` during the live
softpack run. Do not use it as the runtime solver import. The top-level
`newton` package is the live runtime API:

```python
def newton_api():
    import newton

    return newton.ModelBuilder, newton.solvers.SolverVBD
```

The extension may still be needed for a USD/Isaac integration path; enable and
probe that extension separately instead of treating its stub presence as an
import guarantee.

## Newton ModelBuilder soft grid + VBD

The stubs define `ModelBuilder.add_soft_grid(pos, rot, vel, dim_x, dim_y,
dim_z, cell_x, cell_y, cell_z, density, k_mu, k_lambda, k_damp, ...)` and
`ModelBuilder.color(include_bending=False, ...)`. `SolverVBD` is exported by
`newton.solvers` and takes a finalized model plus `iterations` and particle
options. The minimal shape of the working pattern is:

```python
def build_soft_grid_cpu_probe():
    import newton
    import warp as wp

    builder = newton.ModelBuilder()
    builder.add_ground_plane()
    builder.add_soft_grid(
        pos=wp.vec3(0.0, 0.0, 1.0),
        rot=wp.quat_identity(),
        vel=wp.vec3(0.0, 0.0, 0.0),
        dim_x=8,
        dim_y=1,
        dim_z=8,
        cell_x=0.05,
        cell_y=0.05,
        cell_z=0.05,
        density=100.0,
        k_mu=1000.0,
        k_lambda=1000.0,
        k_damp=1.0,
        fix_bottom=True,
        add_surface_mesh_edges=True,
    )
    # REQUIRED before VBD: finalize() does not color implicitly.
    builder.color(include_bending=True)
    model = builder.finalize()
    solver = newton.solvers.SolverVBD(model, iterations=10, particle_enable_tile_solve=False)
    state_0 = model.state()
    state_1 = model.state()
    control = model.control()
    contacts = model.contacts()
    for _ in range(60):
        state_0.clear_forces()
        model.collide(state_0, contacts)
        solver.step(state_0, state_1, control, contacts, 1.0 / 60.0)
        state_0, state_1 = state_1, state_0
    return model, state_0
```

`add_soft_grid` creates a tetrahedral FEM grid and optional surface/bending
edges. `builder.color(include_bending=True)` is **live-verified as required**
for VBD when bending edges are present; omitting it can fail before the first
useful step even though the upstream API documentation does not foreground
the requirement. The snippet is a CPU smoke-probe shape, not a claim that a
CUDA run is safe (see the boundary below).

## The CUDA boundary

Two live findings are operational requirements:

- A PhysX deformable body can fall through the floor if the GPU-dynamics flag
  is lost across `World.reset()`. The pinned run dropped the flag, so set it
  again after every reset and assert it before stepping:

  ```python
  def reassert_physx_gpu_dynamics(world) -> None:
      from isaacsim.core.simulation_manager import PhysxScene

      scene = PhysxScene("/World/PhysicsScene")
      scene.set_enabled_gpu_dynamics(True)
      world.reset()
      scene.set_enabled_gpu_dynamics(True)  # reset drops it in the live image
      if not scene.get_enabled_gpu_dynamics():
          raise RuntimeError("PhysX GPU dynamics was not restored after reset")
  ```

  `PhysxScene.set_enabled_gpu_dynamics` and its getter are stub-proven; the
  reset behavior is **live-verified** for the pinned image.
- The particle-grid VBD path hard-crashed CUDA in the softpack run. Do **not**
  combine a particle-grid soft grid with `SolverVBD` on CUDA. A CPU probe can
  establish authoring and coloring, but this warning is not a suggestion to
  “try one more CUDA setting.” Other solvers or coupling arrangements require
  a fresh live validation and are not promised here.

## USD-schema deformables are a different layer

The `physics.md` capability matrix's older “VBD: no soft bodies” wording was
about an integration layer, not the Newton builder. In the pinned stubs:

- USD/PhysX code uses the deformable-body schema (helper stubs describe it as
  `UsdPhysics.DeformableBodyAPI`; applied schema names include
  `OmniPhysicsDeformableBodyAPI` and the `Physx*DeformableBodyAPI` variants)
  plus the experimental `DeformablePrim`/`SurfaceDeformableMaterial`/
  `VolumeDeformableMaterial` family. Do not assume a standalone Python
  `UsdPhysics.DeformableBodyAPI` class is exported. The old
  `isaacsim.core.api.materials.deformable_material` surface is a removed stub
  that raises; use the experimental material APIs.
- Newton's top-level `ModelBuilder.add_soft_grid` creates soft-grid particles
  and tetrahedra, and `newton.solvers.SolverVBD` supports particle simulation,
  soft bodies, and coupled rigid contacts according to its stub contract.

Choose the layer deliberately. A USD deformable body is not automatically a
Newton `ModelBuilder` particle grid, and a Newton soft grid is not an authored
USD `DeformableBodyAPI` prim. The exact bridge between them is **unverified on
the live image**; keep the two setup and validation paths separate.

## Validation checklist

- Verify the import that will run on the machine (`newton`, not the missing
  `isaacsim.physics.newton` runtime path).
- Call `color()` before `finalize()` whenever VBD will solve particles or
  bodies; use `include_bending=True` when surface bending edges participate.
- Reapply PhysX GPU dynamics after `World.reset()` and assert the getter.
- Keep CUDA particle-grid VBD out of production until a live image proves a
  supported configuration; the pinned softpack result is a hard crash.
- Emit a deformation metric (for example, the maximum particle displacement)
  and a contact/floor metric in the Antioch scenario results instead of relying
  on a rendered frame.
