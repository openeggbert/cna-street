# Visual audit — the state before the overhaul

Written by looking at the eight committed screenshots and at nothing else. The
code is not evidence; the picture is.

Baselines this was taken at:

| Repository | Branch | HEAD |
| --- | --- | --- |
| cna-street | `claude/cna-city-street-demo-ut9lp1` | `c4f80ce` |
| cna | `next` (+ one local commit) | `3471395` (`756096` upstream `next`) |
| sharp-runtime | `next` | `bd282d1` |
| easy-gl | `develop` | `deda7a4` |
| meta-gl | `develop` | `2520173` |

Captured with `--capture docs/visual-overhaul/before --no-overlay` at
1024×576, `--preset high`, on Mesa 25.2 llvmpipe. Scene build 15.9 s,
780 batches, 499 406 triangles, 46 MiB of geometry. All eight unit suites pass
in 1.24 s.

---

## What the pictures actually show

### The two things that destroy it (Priority 1)

**Vehicles are lumps.** `07-the-long-view-south` is the clearest: the green
estate in the foreground has no wheel arches, no visible wheels, no shoulder
line, no glasshouse separation. It is a rounded superellipse tube with a second
rounded tube on top of it, and a quilted normal map is crawling over the whole
thing. At 4 m it does not read as a car; it reads as a car-sized object. The
white ones behind it are worse, because a light body colour hides even the
shut lines.

**People are articulated primitives.** The figure walking towards the camera in
`07` is two cylinders for legs, two for arms, a box torso and a sphere head, and
the arms end in nothing. There is no hand, no shoe, no neck, no shoulder. It is
a mannequin, and the eye finds it instantly — a human silhouette is the shape a
viewer knows best.

### Everything else, roughly in order of damage

3. **The image is washed out.** `02-on-the-crossing` has almost no deep tone
   anywhere: asphalt in full sun sits around 60 % grey, the shadowed façade on
   the left is barely darker than the sunlit one on the right, and the sky and
   the render are close enough in value that the roofline hardly separates. The
   HDR path is technically correct (linear throughout, one sRGB encode) and the
   result is still flat, which means the *values* are wrong, not the pipeline.

4. **Shop interiors are empty white boxes.** `08-shopfronts-close` shows a shoe
   shop — "SCHUHE" on the fascia — containing three grey blocks on a white floor
   against a plywood-coloured back wall. Nothing about it survives being looked
   at from the pavement, which is exactly where a shop window is looked at.

5. **Glass is milk.** The same shot: the glazing is a uniform pale wash with no
   reflection, no fresnel gradient, no frame shadow and no dirt. Real shop glass
   at that angle is mostly a mirror.

6. **Every façade is equally clean.** `01` and `02`: six architectural families
   are visible and all of them are in the same condition, freshly painted, with
   the same dirt map at the same strength. No repairs, no staining under sills,
   no discolouration, no posters, no age difference between neighbours.

7. **Surfaces have no multi-scale detail.** The paving in `01` is a perfect grey
   grid with one crack texture on it. The asphalt has a single aggregate scale.
   The render has one plaster noise. Nothing has both a 2 cm feature and a 2 m
   feature, and real surfaces always have both.

8. **Road markings are new everywhere.** `02` — every zebra stripe has the same
   crisp edge and the same brightness, in the middle of a junction that traffic
   crosses.

9. **Trees are a trunk with lollipops on it.** `01` and `06`: the crowns are
   ellipsoid clusters with a foliage texture on them. The silhouette is a
   cluster of spheres, and no branch structure connects the trunk to the crown.

10. **Edges are all 90°.** Kerbs, window surrounds, string courses, cornices,
    lamp columns — every arris in the scene is a mathematically sharp edge that
    catches no light. Nothing in a built street is sharp.

11. **The far context is unfinished.** `06-above-the-junction` from above shows
    it plainly: past the modelled frontage, buildings become windowless blocks
    with flat grey roofs, and the ground between them is a green-and-grey
    checkerboard. Turning the camera around exposes a stage set.

12. **Street furniture is factory-new.** The bench in `07` is an orange plank
    assembly, the hydrant is a red cylinder with a ball on it, the bin is a grey
    tube. None of them is dented, faded, scratched or dirty at the base.

---

## What is genuinely good and must survive

- The **proportions**. The carriageway, footway, kerb height, storey heights and
  vehicle dimensions are right, and it shows: the street reads as the correct
  *size* even while it reads as CG. `StreetMetrics.hpp` is the reason.
- The **layout and the simulation**. A signalised junction that actually runs,
  traffic that queues for it, a walk graph with kerb waits.
- The **architecture of the renderer**: batching per material and cell, culling
  with separate draw and shadow leashes, instancing, the capability probing.
- The **sky and its IBL**, which is the same sky in the image and in the
  lighting.
- The **content pipeline** and the deterministic screenshot regression.
- The **linear-light discipline**. It is right, and it is the reason the fix for
  the flat image is a values problem rather than a pipeline problem.

## The conclusion this drives

The project's failure is not that it is procedural. It is that the procedural
generators stop at the level of detail where a shape becomes recognisable and do
not continue to the level where it becomes *believable*. A car generator that
lofts a closed tube has made a car-shaped object; one that cuts wheel arches
into the flank and puts a liner behind them has made a car.

So the plan is not "stop generating". It is:

- **generate much harder** where generation is the only option available here
  (see `asset-review.md` — the egress policy in this environment blocks every
  photographic texture library and every model marketplace);
- **import real assets through CNA's real glTF pipeline** where they are legally
  clean and reachable, because exercising that path is one of the reasons this
  application exists;
- and **fix the values**, which is free.
