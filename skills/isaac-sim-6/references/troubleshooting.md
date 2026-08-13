# Troubleshooting — Isaac Sim 6.0.1 on Antioch

Symptom→fix reference. Runs execute remotely. Use the browser stream while a
simulation is live, and use logs, terminal output, telemetry, and artifacts for
evidence that survives it. For run logs and artifact downloads, load the
`antioch-platform` skill.

## Code-level failures (fix in your script)

| Symptom | Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: omni.isaac` | removed shims in 6.0 | rewrite to `isaacsim.*` (map in SKILL.md) |
| Scenario not discovered / import error at discovery | module-scope `pxr`/`omni`/`carb`/`isaacsim`/`isaaclab*` import | move imports into function bodies or `TYPE_CHECKING` |
| Sim runs but objects never move | reading `XformCache` (authored layer) | use `RigidPrim.get_world_poses()` during sim |
| Sensor `get_data()` empty/stale | timeline not committed | `app_utils.play(commit=True)` before first read |
| Velocity set but body doesn't move | wrote `physics:velocity` USD attr pre-play | `RigidPrim.set_velocities(linear_velocities=...)` after `timeline.play()` |
| Angular velocity off by ~57x | `physics:angularVelocity` is degrees/s | convert with `math.degrees()` |
| Physics doesn't advance / jitters | using `app.update()` to step | `World.step(render=True)` or `SimulationContext.step(render=True)` |
| Objects fall through ground | rigid body has no collider under it, or bad bounds from nonuniform scale on an approximated collider | put a `CollisionAPI` prim under the body; validate bounds with collision visualization |
| Intermittent collision failures | collider landed under the wrong ancestor (static instead of part of the body) | colliders resolve to the nearest ancestor with `RigidBodyAPI` — same prim for simple bodies, child prims for compound bodies |
| Wrong rotation downstream | quaternion convention mix | Isaac `[w,x,y,z]` vs scipy `[x,y,z,w]` — convert at boundary |
| `AttributeError: get_rigid_body_state` | removed in 5.1+ | `RigidPrim.get_world_poses()` |
| Articulation moves wrong | solver type | `SolverType=TGS`, position iters >= 6, instanceable asset |
| Free-spinning body slows down | stabilization eats angular momentum | `EnableStabilization=False`, zero damping/joint friction |
| Cradle/chain momentum doesn't propagate | PhysX same-island limit (permanent) | script analytical transfer or use Newton |
| Contact chain tunnels at high spin | compound colliders > ~50 rad/s | convex hulls or physics Hz > 480 |
| RL agent fights the drives | PD drives left active | zero stiffness/damping, or no DriveAPI authored |
| Newton + torch hang at startup | CUDA context conflict | import torch only after `timeline.play()` + settle |
| Unexpected physics behavior | Newton switched on explicitly earlier | check `SimulationManager.get_active_physics_engine()` — enabling the extension alone only registers it |

## Run-level failures (fix in scene/config, or wait)

| Symptom | Cause | Fix |
|---|---|---|
| First step takes 5–15 min | cold MDL shader compile | normal on cold start; wait — warm machines cache it |
| Boot crawls on a high-core host | thread over-subscription | cap workers via `antioch.boot(extra_args=["--/plugins/carb.tasking.plugin/threadCount=16"])` |
| Hang at first `play()` / reset | no `/World/PhysicsScene` defined | define it before playing |
| Hang at first step | contact storm: too many initial contact pairs | enable stabilization; separate overlapping spawns |
| Black render artifact | no lights, or wrong render mode | DomeLight (>= 100) + DistantLight (>= 500); `RaytracedLighting` for iteration, PathTracing only for hero shots |
| Flat, shadowless render | low pixel variance | accent lighting; check light cone bounds |
| Replicator capture hangs | timeline not playing / no fresh frame | play timeline; step once before capture |
| `rep.orchestrator.run()` returns with no dataset files | managed-run `run()` only submits an asynchronous start | Capture with `rep.orchestrator.step(...)`, drain `rep.backends.io_queue.wait_until_done()`, then check the writer count before archiving; use `await step_async()` only inside an async context |
| Stage open takes minutes | > 1000 layers | consolidate into library layers |
| Slow asset load / resolver stalls | missing external references | audit `layer.GetExternalReferences()` |
| CUDA OOM | too many envs/bodies for VRAM | reduce num_envs; instanceable assets; skip appearance payloads |
| Render OOM | texture streaming budget | lower `/rtx-transient/resourcemanager/maxMipCount`; reduce resolution |
| Sim "passed" but results wrong | < 3 s of physics | run >= 3 s settle before reading final state |

## Render artifact validation (remote QA gate)

You cannot eyeball frames live — validate the artifact programmatically after
the run:

| Metric | Good | Bad | Action |
|---|---|---|---|
| File size | 1–4 MB | ~82 KB | 82 KB ≈ black frame; check lights |
| Mean RGB | 80–160 | < 20 | too dark; add lights |
| Max RGB | 200–250 | 0 | no light in scene |
| Pixel variance | > 15 | < 5 | flat = missing shadows |

ACES tonemap (`/rtx/post/tonemap/op=6`) with `filmIso` ~200 default, 600 for
deep-aisle warehouse shots, 400 aerial.

## Physics hang triage

1. Is `/World/PhysicsScene` defined before play? (most common)
2. How many contact pairs on step one? Separate initial overlaps.
3. GPU dynamics with > 100K rigid bodies? Reduce or instance.
4. Newton active unexpectedly? Check the active engine before debugging PhysX
   settings that aren't being read.
