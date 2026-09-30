# Navigation and capture

`antioch.lib.navigation` moves a native inspection camera; `antioch.lib.viewport`
reads the existing viewport. Neither advances physics or moves the subject.

## Position the view

```python
from IPython.display import display
from antioch.lib import navigation, viewport

navigation.look_at(eye=(4, 4, 3), target=(0, 0, 0.5))
display(await viewport.observe_async())
```

More operations, none of which render:

```python
saved = navigation.get_pose()
navigation.translate((0, 0, -1), space="camera")  # forward along local -Z
navigation.translate((0, 0, 1), space="world")
navigation.rotate("y", 15, space="camera")  # degrees
navigation.rotate((0, 0, 1), 30, space="world")
navigation.orbit((0, 0, 0), azimuth_deg=45, elevation_deg=20, radius=4)
navigation.frame_object("/World/Robot", margin=1.2)
navigation.set_pose(saved)
```

Distances use stage units; orientations use USD WXYZ quaternions. Orbit angles
are absolute; an omitted radius keeps the distance to the pivot. `frame_object`
fits authored USD bounds; query live simulator state to locate moving bodies.

Mutations default to the inspection camera `/OmniverseKit_Persp`; each helper
accepts `camera_path`. To inspect an authored sensor without changing its pose
or lens, select it:

```python
navigation.select_camera("/World/Camera")
display(await viewport.observe_async())
```

## Read pixels

Keep navigation and capture in the same cell. Use `observe_async()` in Jupyter
and `observe()` in synchronous simulator code. `display(frame)` shows it;
`frame.image` holds PNG bytes; `frame.metadata` reports status, camera,
dimensions, capture times, and timeline positions. The timeline can move during
render-only updates, so measure physical progress from the physics clock or
body state. An unavailable capture returns no image; read the status instead
of reusing old pixels. Inspection images fit within 1280×720; for full
resolution use `capture_viewport_async()` or its synchronous counterpart.

Each readback has a ten-second deadline, which cannot interrupt a render in
progress: the first frame a process renders has held a cell for up to five
minutes while the renderer filled its cache. If capture is unavailable during
startup, let the render loop progress and retry rather than replaying scene
creation.

Capture uses the active native viewport, not a new render product, and proves
neither shader convergence nor correct scene content; inspect the image.

Use [Jupyter](jupyter.md) for persistent inspection, [native camera guidance](../../isaac-sim-6/references/isaac-camera.md) when the camera is part of the sensor model, and [telemetry](../../scenario-design/references/telemetry.md) to retain selected images with a run. Return to [agentic simulation](../SKILL.md#measure-the-result) to choose evidence for the claim being tested.
