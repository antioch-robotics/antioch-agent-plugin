# Sensors and cameras

Use this for camera, lidar, radar, acoustic, IMU, contact, and joint-state
sensing. All simulator imports are lazy and run after Kit starts.

## Select the surface

| Need | Pinned API entry point |
|---|---|
| Camera authoring and capture | `isaacsim.sensors.experimental.rtx`: `RtxCamera`, `CameraSensor` |
| Batched cameras or single-view depth | Same module: `TiledCameraSensor`, `SingleViewDepthCameraSensor` |
| RTX lidar, radar, acoustic sensing | Same module; retrieve the authoring/runtime classes and supported model configuration |
| IMU, contact, joint state, raycasts | `isaacsim.sensors.experimental.physics` |
| Current viewport image | `antioch.capture_viewport()` |
| Replicator-specific annotators/writers | `omni.replicator.core` |

Enable required extensions through `SimulationConfig.extensions` before
startup. Do not assume every installed extension is enabled by the experience.

Some classic sensor APIs remain shipped but deprecated. Check the exact
symbol; do not claim that every old namespace was removed or replace a
working integration without a reason.

## Authoring versus runtime

A USD sensor prim defines pose and configuration. A runtime sensor owns data
production, buffers, and annotators. For the experimental camera stack, pose
the `RtxCamera` or its USD prim; `CameraSensor` is not a transform wrapper.

Construct the scene, attach the sensor, initialize/play through the owning
framework, and render or step as the sensor requires. Then wait for a fresh
sample with a bounded deadline. Ten update calls do not guarantee readiness.
For `CameraSensor.get_data`, handle the returned `(data, info)` and an absent
`data` before conversion. Validate timestamp and expected shape, not just
non-nullness. Respect buffer lifetime when retaining a sample across updates.

A `JointStateSensor` targets an articulation, not an arbitrary joint prim.
For contact sensors, define the measured body and filter relationships
explicitly. For IMUs, verify reference frame, gravity treatment, and units.

## Camera units and axes

USD cameras look along local -Z with +Y up, independent of stage up-axis.
Isaac Sim's quaternion APIs use WXYZ unless the chosen API states otherwise;
Isaac Lab 3 and many external libraries use XYZW.

Raw `UsdGeom.Camera` focal length and apertures use **tenths of a scene unit**.
On a meter stage, 24 mm is `0.24`. Focus distance and clipping range instead
use scene units. Higher-level wrappers may expose different units; inspect
their setters rather than copying raw USD values into them.

| Capture API | Resolution input | Image layout |
|---|---|---|
| Experimental `CameraSensor`, `TiledCameraSensor`, `SingleViewDepthCameraSensor` | `(height, width)` | Image axes are height, width; tiled data also has batching |
| `rep.create.render_product` | `(width, height)` | Image annotators use height, width |

For a centered pinhole camera with pixel focal lengths `fx, fy` and image
size `W, H`, choose a physical focal length `f`, then set physical apertures
`A_x = f * W / fx` and `A_y = f * H / fy`. Convert all three physical lengths
to the raw USD optical units. Do not compute vertical aperture without image
height. For a displaced principal point, verify aperture-offset signs against
a projected calibration target and the image's coordinate convention.
Lens distortion requires the appropriate camera model, not just a pinhole
matrix copied into arbitrary attributes.

The unit contract is documented by
[OpenUSD Camera](https://openusd.org/release/api/class_usd_geom_camera.html).
Check the renderer/wrapper's supported distortion and shutter behavior at the
runtime pin.

## Viewport versus dedicated camera

Creating a camera prim does not select it as the viewport camera. For the
classic World path, aim the active viewport after reset and render before
readback. `/OmniverseKit_Persp` is a common active camera path, not a guarantee.

`antioch.capture_viewport()` returns a frame or `None`. The SDK can use the
window viewport or an offscreen render product, depending on the app state;
it is not guaranteed to avoid creating a render product. Use dedicated sensor
products when the task needs fixed calibration, multiple views, or specific
annotators.

## RTX lidar, radar, and acoustic data

Retrieve the supported configuration for the selected sensor model. Creating
a sensor at a child path does not establish the correct mount transform,
scan pattern, firing time, return layout, or noise model.

Check sensor-to-robot and sensor-to-world transforms, units, scan period,
valid-return flags, and timestamp alignment. Decode the actual annotator
schema; a generic point cloud assumption is not enough for radar or acoustic
payloads. Keep writer/buffer ownership and device-to-host copies explicit.

Validate against known geometry: a plane at a measured distance, a known
angular target, or a controlled motion. A nonempty buffer alone does not
prove calibration or a fresh scan.

## Output checks

- Decode the saved image or array; check shape, dtype, and relevant finite
  values. Depth backgrounds may legitimately use nonfinite values, so define
  the valid-pixel mask from the annotator contract.
- Check task content: target pixels, projected geometry, labeled objects,
  measured range, contact response, or motion-correlated readings.
- Keep failing samples as diagnostics and record a failed check. Do not hide
  a bad frame by refusing to log it.
- Detach owned annotators/writers and release owned render products in cleanup.
  Do not destroy the platform's viewport or another sensor's product.

`references/rendering.md` covers lighting and image diagnosis.
`references/sdg.md` covers datasets and writer completion.
