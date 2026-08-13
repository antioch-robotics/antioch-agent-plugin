# Lighting and tone-mapping recipes

Complete code for the tables in `SKILL.md`. All functions run inside a
scenario after the runner has booted Kit — they never call `antioch.boot()`
themselves. Lazy imports in function bodies only.

## Baseline lights

An authored stage has no default lighting. Without explicit lights every frame
is black. This is the minimum viable setup:

```python
def add_baseline_lights(dome_intensity: float = 400.0, sun_intensity: float = 1500.0) -> None:
    from pxr import Gf, UsdGeom, UsdLux

    import antioch

    stage = antioch.stage()
    dome = UsdLux.DomeLight.Define(stage, "/World/DomeLight")
    dome.GetIntensityAttr().Set(dome_intensity)
    sun = UsdLux.DistantLight.Define(stage, "/World/Sun")
    sun.GetIntensityAttr().Set(sun_intensity)
    UsdGeom.Xformable(sun.GetPrim()).AddRotateXYZOp().Set(Gf.Vec3f(-50, 20, 0))
```

| Scene type | DomeLight | DistantLight | Notes |
|---|---|---|---|
| General interior | 400 | 1500 | Good general balance |
| Close-up subject | 300 | 1200 | Slightly softer |
| Outdoor | 500 | 2000 | Brighter sun |
| Dark / moody | 100 | 800 | Dramatic shadows |

## ACES tone mapping

The single biggest quality lever after the lights. Without ACES, no intensity
tuning produces a balanced enclosed render.

```python
def enable_aces(film_iso: float = 200.0) -> None:
    import carb

    s = carb.settings.get_settings()
    s.set("/rtx/post/tonemap/op", 6)  # ACES (4 is HejlHableAlu)
    s.set("/rtx/post/tonemap/filmIso", film_iso)
    s.set("/rtx/post/aa/op", 1)  # TAA (3 = DLSS, which also upscales)
```

filmIso calibration (validated on enclosed interiors, RaytracedLighting):

| Scene | filmIso |
|---|---|
| General interior | 200 |
| Aerial / overview-heavy | 400 |
| Deep-occlusion interiors | 600 |

Anti-recipes, all validated as wrong:

- Wide rect lights (width 5+) → flat, no light pools.
- Dome intensity 400+ with filmIso 600 → washed-out shadows.
- Reinhard tone mapping (`/rtx/post/tonemap/op` other than 6) → muddy, low
  contrast.
- PathTracing while iterating → 5–30 min per frame.

## Quality recipe: light pools + accent asymmetry + fog

The jump from "lit" to "photoreal" for large interiors comes from three
moves: keep ambient low so local lights do the work, use narrow rect lights
for pools of light instead of floods, and add subtle fog to sell scale.

```python
def add_photoreal_interior_lights(fixture_positions: list, accent_positions: list) -> None:
    """Low cool dome + focused rect pools + alternating warm accents."""
    from pxr import Gf, UsdLux

    import antioch

    stage = antioch.stage()
    dome = UsdLux.DomeLight.Define(stage, "/World/L/Dome")
    dome.CreateIntensityAttr(150.0)  # low — let spots do the work
    dome.CreateColorAttr(Gf.Vec3f(0.85, 0.88, 0.95))  # cool blue-ish

    for i, pos in enumerate(fixture_positions):
        rl = UsdLux.RectLight.Define(stage, f"/World/L/Pool{i}")
        rl.CreateIntensityAttr(12000.0)
        rl.CreateWidthAttr(1.5)  # narrow: pools, not floods
        rl.CreateHeightAttr(0.2)
        rl.CreateEnableColorTemperatureAttr(True)
        rl.CreateColorTemperatureAttr(4200.0)  # neutral-warm
        # position + aim down per fixture_positions[i]

    for i, pos in enumerate(accent_positions):
        # alternate left/right, warm color, mid-height — asymmetry reads as real
        pass


def enable_fog(density: float = 0.004) -> None:
    """Subtle cool fog; ~0.003 for a 40 m space adds depth without murk."""
    import carb

    s = carb.settings.get_settings()
    s.set("/rtx/fog/enabled", True)
    s.set("/rtx/fog/fogDistanceDensity", density)
    s.set("/rtx/fog/fogColor", (0.85, 0.87, 0.92))
```

## Deep-occlusion interiors: multi-layer lighting

Problem: a ground-level camera in a narrow aisle (e.g. 3.5 m wide, 8 m tall
shelving) renders black (mean RGB < 5) even with strong ceiling lights — RaytracedLighting
cannot carry light down through deep occlusion.

Solution: layers at every height, plus a longer RTX render-history warm-up (see
`references/rendering.md` for the convergence probe).

```python
def add_aisle_lights(ceiling_grid: list, aisle_positions: list) -> None:
    """Validated at ACES filmIso=600 with NO dome light."""
    from pxr import UsdLux

    import antioch

    stage = antioch.stage()
    for i, (pos, aim) in enumerate(ceiling_grid):
        rl = UsdLux.RectLight.Define(stage, f"/World/L/Ceil{i}")
        rl.CreateIntensityAttr(70000.0)  # validated
        rl.CreateWidthAttr(2.5)
        rl.CreateHeightAttr(1.5)
        rl.CreateColorAttr((1.0, 0.97, 0.92))  # warm white
        # place at ceiling z - 0.3, pointing down

    for i, pos in enumerate(aisle_positions):
        lt = UsdLux.SphereLight.Define(stage, f"/World/L/Aisle{i}")
        lt.GetRadiusAttr().Set(0.1)
        lt.GetIntensityAttr().Set(15000.0)  # validated
        # place at z ~ 3.5 (head height), inside the aisle
```

Validated configuration (enclosed warehouse-scale interior, RaytracedLighting, filmIso 600):

- Ceiling rect grid 8×14, intensity 70,000, 2.5 × 1.5 m, warm white.
- Sphere light per aisle, intensity 15,000, radius 0.1, at Z ≈ 3.5 m.
- No dome light — ACES handles exposure.
- Start with 300–500 `update_app()` warm-up frames for this deep-occlusion
  shot, then keep the count only if denoiser/temporal frame statistics are
  still changing; 500 is a budget, not a magic constant.
- Result: mean RGB 60–155 across hero / overview / cross-aisle views.

Higher-output variant when 70K/15K still underexposes: ceiling grid at
200,000 with wider 4.0 × 3.0 panels, aisle spheres at 100,000 with radius
0.15, optional dome at 300 as ambient fill — never higher, or open views
wash out.

### The dome-vs-aisle exposure tension

In enclosed scenes one global setting cannot serve every view:

- High dome → overview and top-down shots overexpose (mean > 220).
- Low or no dome → deep aisles underexpose (mean < 10).

Resolve it with local light, not ambient: no dome, aisle-level sphere lights,
dense ceiling grid, and a render-history warm-up selected by the frame-stat
stability probe (often 300–500 updates). Expected means by view type: hero
aisle ~60, elevated three-quarter overview ~140–175, cross-aisle ~230.

### Camera placement for interiors

Put hero cameras at aisle intersections, not deep inside narrow aisles.
Junctions have open volume on multiple sides, so light reaches the subject
without pushing intensities into overexposure elsewhere.
