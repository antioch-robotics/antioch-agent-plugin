# Mechanism experiments

Adapted from NVIDIA [`physics-simulation/examples.md`](https://github.com/isaac-sim/IsaacSim/blob/7c206f75bdadd9e05fc457f19863ca4c3f0cb693/skills/physics-simulation/examples.md) (Apache-2.0).

[Isaac Sim task index](../SKILL.md#domain-references) · [Antioch startup](../../antioch-platform/references/simulation-code.md)

## Worked Examples

These five upstream mechanisms show different experiment structures. Their dimensions and solver settings are starting points, not acceptance criteria or guarantees. Start with [physics configuration](physics-simulation.md), use one stepping owner, and record enough live state to explain the result. Native code fragments run after [Antioch startup](../../antioch-platform/references/simulation-code.md).

### 1. Impact / Crash (Vehicle vs Bollard)

Compare a rigid mount with a compliant mount, then measure where the impact energy goes.

**Starting experiment:** use CCD when tunneling is a risk, then compare timestep and solver-iteration settings until the measured result converges. The upstream template starts at 120 Hz with TGS 32/8; verify the actual stepping configuration rather than assuming extra render updates create substeps.

**Two architectures:**
1. **Fixed bollard:** a static mount transfers the impact through contact.
2. **Spring-return bollard:** a prismatic joint with a spring and damper allows displacement and dissipates energy. Measure travel, rebound, and impulse; the drive gains and limits determine the response.

**Velocity injection:** set linear velocity via experimental `RigidPrim.set_velocities(linear_velocities=...)`
**after** play/reset through the stepping owner. The USD `physics:velocity` attribute is authored, not
runtime — this is documented in Part 8 of [physics-simulation.md](physics-simulation.md).

**Analysis:**
```
Impact energy:  E = 0.5 * m * v²        # 2000 kg @ 8 km/h = 4938 J
Average force:  F_avg = Δp / Δt  # over the sampled interval, not peak force
Restitution:    e = v_rebound / v_impact
```

Use contact samples to measure peak force at their actual sample interval. The upstream experiment reported about 65 kN peak for the rigid mount, and about 95% energy absorption, less than 0.5 m vehicle rebound, and over 90% lower peak force for its spring mount. These are source-specific observations, not validated Antioch results or predictions for another bollard.

The source names ASTM F2656, PAS 68, and IWA 14-1 as barrier-test references. Check the applicable standard and its current requirements separately; simulation output alone does not establish a certified rating.

### 2. Vibratory Bowl Feeder (kinematic high-frequency animation)

Drive the bowl at physics-step cadence so the renderer does not change its excitation. Compare phase, amplitude, and frequency against measured part transport.

The upstream sketch simplifies bowl motion to translation along X and Z. It does not model torsional bowl motion along a helical track; use that geometry and motion explicitly when they matter to the feeder.

**Upstream translation parameters (starting points):**
- Frequency 60 Hz, vertical amplitude 0.3 mm, X amplitude 0.1 mm
- Phase offset: horizontal motion leads vertical by π/4 in this starting experiment; geometry, friction, and contact determine whether parts climb

The upstream example uses 480 Hz, eight samples per 60 Hz cycle, with CCD enabled, 16/4 solver iterations per part, and stabilization disabled. Keep these as the source's experiment settings. Test finer timesteps for convergence; inspect contact and damping before adjusting solver settings.

```python
import math


def bowl_offset(sim_s, hz=60.0, amp_z=0.0003, amp_x=0.0001):
    """Translation offset for a driven bowl, in meters."""
    omega = 2 * math.pi * hz
    return (amp_x * math.sin(omega * sim_s + math.pi / 4), 0.0, amp_z * math.sin(omega * sim_s))
```

Apply the offset relative to the bowl's initial pose through the selected backend's kinematic-body API on each physics step. Preserve its orientation and scale. Confirm how that API derives surface velocity from kinematic targets, since contact transport depends on it. Measure part displacement and contacts; the bowl moving as requested is not evidence that it carries parts correctly.

### 3. Spinning top

A spinning top tests inertia, damping, gravity, contact, and numerical stability together. Start with the authored mass, inertia tensor, and center of mass. A thin disk has axial inertia `Izz = 0.5 * m * r**2`; a solid cone has `Izz = 0.3 * m * r**2` about its symmetry axis.

For fast, approximately steady precession, `precession_rate ≈ m * g * d / (Izz * spin_rate)`, where `d` is the pivot-to-center-of-mass distance. This approximation does not decide whether every initial condition is stable. The upstream ratio `Izz * spin_rate / (m * g * d * sin(tilt))` has units of time, so it is not a dimensionless stability threshold.

For an ideal symmetric top upright on a fixed pivot, the small-tilt stability condition is `Izz**2 * spin_rate**2 > 4 * I_perp * m * g * d`, where `I_perp` is the transverse inertia about the pivot. The ratio of the two sides is dimensionless. This is the [sleeping-top result](https://mitp-content-server.mit.edu/books/content/sectbyfn/books_pres_0/9579/sicm_edition_2.zip/chapter003.html), not a guarantee for a sliding tip, large tilt, or dissipative contact.

Measure orientation, spin, angular momentum, and contact through time. Compare timesteps and solver settings. Simplify compound colliders when diagnosing tunneling, but retain the geometry needed for the task. The upstream top reportedly tunneled at about 50 rad/s after 5–6 s; that observation is specific to its compound colliders and setup. A speed threshold observed on one top is not a general backend limit.

### 4. Newton's cradle and contact chains

Use a pendulum chain to examine sequential impulse transfer. Record each body's trajectory, contact times, impulse, and energy rather than judging only the final render. Start with known masses, joint lengths, contact geometry, and initial displacement.

The upstream example reported all spheres moving together under several PhysX configurations. It tried TGS and PGS at 64/32 solver iterations, 120/240/480 Hz, 0.1–0.2 mm initial gaps, restitution 1.0 with `restitutionCombineMode=max`, and CCD. Those are the source's reported experiments, not proof that every same-island contact chain is impossible. Test initial gaps, timestep, restitution, collision approximation, and solver settings in a small case, and retain the run evidence.

Do not replace the chain with baked animation and call it physics. If the requested model uses analytical impulse transfer, label that part as scripted. Contact evidence comes from the contact or impulse API, not only proximity.

### 5. Escapement Clock — Hybrid Physics + Scripted Mechanism

An escapement can be modeled through contact or through a hybrid controller, depending on the task. The hybrid below makes the pendulum dynamic and the mechanism state-driven. Choose it when that abstraction is acceptable; there is no universal 10 mm cutoff for physical contact simulation.

**Architecture:**

| Component | Mode | Role |
|---|---|---|
| Pendulum | **Dynamic** rigid body, revolute joint, gravity-driven | The real physics |
| Escape wheel | **Kinematic**, rotation set by controller | Scripted mechanism |
| Anchor | **Visual-only** (rotation derived from pendulum angle) | Cosmetic |
| Frame | **Static** | Housing |

**4-state mechanism controller:**
```
LOCKED_LEFT → RELEASING_RIGHT → LOCKED_RIGHT → RELEASING_LEFT → LOCKED_LEFT
```
- `LOCKED_*`: wait for pendulum to swing past `RELEASE_ANGLE`
- `RELEASING_*`: pendulum returns past `LOCK_ANGLE` → kinematic wheel advances by
  `HALF_TOOTH_DEG` (e.g., 18° for a 10-tooth wheel) → next `LOCKED_*` state

One full tick = two half-ticks = one tooth.

**Pendulum configuration to inspect:**
- `enableStabilization=False`, `angularDamping=0.0`, `jointFriction=0.0`
- Author body inertia about its center of mass. `m_bob * L**2 + m_rod * L**2 / 3` approximates inertia about the pivot for a point bob and uniform rod; it is not the USD diagonal inertia to assign to the body.
- Solver iterations and timestep, checked for convergence
- Collision filtering that matches the model; fix unintended frame intersections rather than hiding required collisions

**Initialization:** set the pendulum's initial pose or joint state through its owning API, then reset and step. Measure its free-decay behavior before adding a sustaining drive. If a tick applies an impulse to replace lost energy, log that input and describe the model as driven. Do not hide impulses to make passive behavior look correct.

The upstream demonstration used 32/16 solver iterations, a 7° initial tilt, no collision on the bob, and angular-velocity increments of about 0.015 rad/s at each tick. It reported about 40% energy loss over 15 s before compensation. These observations describe that setup; they do not establish a universal damping rate. Disabling bob collision is only appropriate when bob contact is outside the intended model.

**Reusability:** this controller advances the wheel when the pendulum crosses angle thresholds. Another mechanism can use a measured pose, joint angle, or contact event as its trigger; state which signal drives each transition.

---

## Integration

- Robot USD / drives: [urdf-mjcf-to-usd-conversion.md](urdf-mjcf-to-usd-conversion.md)
- Multi-link assembly: [usd-articulation.md](usd-articulation.md)
- SDG scenes: [data collection](data-collection-sim.md)
- Grasping physics: [manipulation-ik.md](manipulation-ik.md)
- Hang/freeze: [isaac-sim-troubleshooting.md](isaac-sim-troubleshooting.md)
- Scene/backend contract: [physics-simulation.md](physics-simulation.md)

Use [scenario design](../../scenario-design/SKILL.md) to turn each experiment into parameterized inputs, measured checks, and saved telemetry.
