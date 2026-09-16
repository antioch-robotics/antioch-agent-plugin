# Navigation and capture

`antioch.lib.navigation` controls a native inspection camera without
quaternion calculations in task code. `antioch.lib.viewport` reads the
existing viewport. Neither helper advances physics or moves the subject.

## Position the view

```python
from IPython.display import display
from antioch.lib import navigation, viewport

navigation.look_at(eye=(4, 4, 3), target=(0, 0, 0.5))
display(await viewport.observe_async())
```

Useful operations compose without rendering:

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

Distances use stage units; orientations use USD WXYZ quaternions. Orbit
angles are absolute; omitted radius preserves distance to the pivot.
`frame_object` fits authored USD bounds with a perspective camera; query
live simulator state to locate moving bodies.

Mutations default to the inspection camera `/OmniverseKit_Persp`.
Each helper accepts `camera_path` for an explicit target. To inspect an
authored sensor without changing its pose or lens, select it:

```python
navigation.select_camera("/World/Camera")
display(await viewport.observe_async())
```

## Read pixels

Keep navigation and capture in the same cell. Use `observe_async()` in
Jupyter and `observe()` in synchronous simulator code. Display the returned
observation with `display(frame)`; `frame.image` contains encoded PNG bytes
for saving. `frame.metadata` reports status, camera, dimensions, capture times,
and animation timeline positions.
The timeline can move during render-only updates; use the simulator's physics
clock or body state to measure physical progress.
Unavailable capture returns no image; inspect the status instead of reusing
old pixels. Inspection images fit within 1280×720. For full-resolution RGB
use `capture_viewport_async()` or its synchronous counterpart.

Each readback has a ten-second deadline, separate from native scene loading.
If unavailable during startup, inspect loading status and errors, let the
existing render loop progress, then retry capture within the experiment budget.
Do not replay scene creation or robot actions to retry an image.

Capture uses the active native viewport, not a new render product. Readiness
does not establish shader convergence, correct scene content, or a connected
browser stream. Inspect the image and task state. Save important frames to
project files or scenario artifacts before session retirement.
