# The second visual pass: what was wrong, what changed, what it cost

A third pass over `cna-street` with the rendered result as the acceptance
criterion, starting from the first overhaul's own honest list of what still
gave the street away. Written against the frames in this directory; the code
is evidence of what was done, the pictures are evidence of whether it worked.

Baselines, recorded before anything changed:

| Repository | Baseline | Note |
| --- | --- | --- |
| `cna-street` | `27f92a8` on `develop` | The commit this pass starts from |
| `cna` (`../cnanext`) | `next` `34c5a9d` at the start, `e1d3aa5` by the end | Read-only; advanced under another agent during the session |
| `sharp-runtime` (`../sharp-runtimenext`) | `next` `c3fbb95` | Read-only |
| `easy-gl` | `develop` `deda7a4` | Read-only |
| `meta-gl` | `develop` `20c8b2d` | Read-only |

**No framework repository was modified.** The session began by discovering that
`../cna` is on `develop`, which carries none of the `CNAEXT` engine layer this
project is built on, and the user pointed it at `../cnanext` (`next`) and
`../sharp-runtimenext`; `dependencies.lock` and the README now say so. Every
CNA behaviour this pass relied on or ran into is in `docs/cna-findings.md`
under *Second visual pass*.

---

## 1. What was wrong with the current render

Rendered before changing anything, all fourteen viewpoints, day and night, and
looked at rather than reasoned about. In order of visible damage:

1. **Cars were shapes, not cars.** A black car was a dark silhouette with no
   horizon on it; a white one was a flat white lump. Both because the only
   environment anything reflected was the sky. The tail lamps were red slabs
   the width of the boot standing 14 cm proud of the bumper, on every car,
   braking or not.
2. **Shop windows reflected nothing and showed a white box.** The pane was a
   pale coloured filter over an interior lit to the brightness of the pavement,
   with two continuous light strips across the ceiling and a shelf of
   pink-and-green confetti. The pillars between the panes read as concrete.
3. **The far end of the street was a set.** Beyond 130 m the highway stopped,
   the ground became a flat plane, and the blocks were boxes with a picture of
   a storey on them.
4. **Every rendered wall wore the same dark dashes.** The plaster's crack field
   put a black stroke every few centimetres on every render facade in the
   city, which from the footway read as flies on the paintwork.
5. **The night sky was brown.** The scattering model's residual term
   everywhere, with pale brown cloud blobs in it, and a street too dark to
   read.
6. **The street tree viewpoint showed a facade**, because it was aimed at
   where a tree was expected.

Behind all of those, one architectural gap: a renderer with a sky cube and
nothing else for reflections, and a transparent phase whose blend state
multiplied every reflection by the pane's alpha.

## 2. What changed

**Reflection probes.** Twenty-nine points along the carriageways -- a row over
each parking lane at the height of a car door, a row over each side-street
lane, one at the junction. At scene build the static street is rendered six
ways from each into a 64 px `RenderTarget2D`, read back, decoded from the
effect's own sRGB encode, and written into a `TextureCube` at the sky cube's
scale; `EnvironmentProcessor` then prefilters it exactly as it does the sky,
and generates its irradiance too. Every static batch and every vehicle reads
its `ImageBasedLightEXT` from the nearest probe; instanced props and people
keep the sky's. The cascades are re-fitted once per probe from a camera
looking straight down at it from 55 m, so one shadow pass serves all six
faces. Seven seconds at start-up, no measurable cost per frame; re-baked when
the sun stops moving. The face basis is pinned by a test against the cube
layout, because a face captured mirrored is a reflection of the wrong side of
the street and nothing reports it.

**Glass.** `Material::premultipliedBlend`: glass asks for XNA's `AlphaBlend`
per draw inside the pipeline's transparent phase, so it composites as
`lit + behind * (1 - alpha)`. The base colours went from pale to near black
(the tint of the reflection layer), the alphas from 0.34/0.24/0.80 to
0.22/0.14/0.55 (how much the pane blocks). The image-based intensity returned
to 1.0, because the 0.86 stood in for a canyon's missing sky and the probes now
capture the canyon.

**Shops.** Walls tinted to the trade in a front and a back shade, the ceiling a
third darker again; tube fittings a hand wide in 1.5 m lengths; a poster atlas
and posters hung where each trade hangs them; a packaging texture -- runs of
white card with a label band, a few coloured boxes, a few dark ones -- in place
of the hue-wheel confetti; one unit in six shut behind a roller blind; and
every interior light level roughly halved. Interiors are flagged `sunlit =
false`, so the sun and its 8-bit acne never reach the back wall of a room.

**Vehicles.** The lamp strip wrapped toward the middle of the car from either
end -- it used to wrap 26 cm out behind the rear bumper. Brake lenses are the
same strips three millimetres over the tail lenses. Tail clusters a hand tall,
headlamps with a chromed reflector unit inside a dark housing, a grille with
three bright slats, a frame and a badge, a roof aerial.

**Facades.** A weathering-decal atlas -- sill run-off, plinth splash, downpipe
stain, soot -- laid where the water actually runs, with the probability and
the length of each scaled by the plot's weathering value; the render's cracks
cut to one every few metres and only on weathered plots; planters and chairs
on the balconies; satellite dishes with their cables.

**The district beyond.** Carriageway, footways and kerb faces continued to the
closing block; the forty blocks lining the two streets given real window
recesses with a reveal, a dark room, a pane and a sill each, shopfronts under
fascias, plinths and cornices; cross streets every third block; the far tree
level of detail on the footways and the distant car bodies in the bays.

**Night.** A hand-added twilight blue with the afterglow kept to the sun's
azimuth; clouds that take the sky's own colour once the sun is down; exposure
1.5.

**The street.** Bicycles on the stands, collars on the coats, caps on one head
in seven, six greens for six trees, the tree viewpoint aimed at a tree, and
two photographic viewpoints.

## 3. CNA capabilities used, newly or more fully

- `RenderTarget2D` as a capture target with `GetData` readback, and
  `GraphicsDevice::SetRenderTarget` / `Clear` around it, for the probe faces.
- `TextureCube::SetData` per face to assemble a probe cube on the CPU.
- `EnvironmentProcessor::generatePrefilteredSpecular` and
  `generateIrradiance` on something other than the sky.
- `PbrEffect::setImageBasedLightEXT` rebound per draw, which the effect
  handles correctly.
- `CascadedShadowMap::update` with a fitting camera that is not the viewing
  camera, relying on `ShadowCascadeStateEXT::CameraView` for the receiver's
  cascade selection, exactly as that field documents.
- `BlendState::AlphaBlend` set inside `RenderPipeline::setTransparentScene`'s
  callback, which the pipeline honours.
- `Matrix::CreateLookAt` / `CreatePerspectiveFieldOfView` and
  `BoundingFrustum` for the per-face culling.
- `MaterialLibrary::deriveTinted` -- this project's own, over CNA's material
  model -- used far more: shop tints per trade, foliage per tree variant,
  bicycle paints.

Nothing was called to be able to list it. Screen-space reflections stayed
off, for the reasons the first overhaul recorded.

## 4. Tried and rejected

- **Probe irradiance for the walls at the old intensity.** With the sky's
  0.86 ambient stand-in still in place the whole street went a stop darker,
  because the probes already carried the canyon's occlusion. Fixed by
  restoring 1.0 rather than by scaling the probes.
- **Diagnosing horizontal bands on the shop back wall as a post-process.**
  Renders without light shafts, bloom, SSAO, MSAA, FXAA, fog and HDR all kept
  them. They went away with the interior rework, whose walls no longer stretch
  one unit UV across four metres; the cause was the texture, not the pipeline.
- **Zero-context hunk staging.** Not a rendering experiment, but worth a line:
  splitting the shared files into themed commits with `git diff -U0` mis-placed
  hunks, and the commits had to be redone from ordinary context diffs with the
  entangled edits moved apart in the source first. Every commit in this pass
  builds and passes the tests on its own.

## 5. CNA findings

No defect. Five environment and behaviour notes, in `docs/cna-findings.md`
under *Second visual pass*: `develop` lacks the engine layer; the transparent
phase's blend state is a default a draw may override; `ImageBasedLightEXT` may
change per draw; the cascade fit is perspective-only, and how to fit for a
point; `Color` is the only render-target readback. CNA was not modified.

## 6. Performance

See `performance.md` for the method and the table. Measured with `--frames 26`
at 1024 × 576, viewpoint 1, the two builds interleaved in one session on a machine
that was busy with other work throughout.

Against `27f92a8`, eight interleaved pairs at 1024 × 576: median of medians
40.6 ms before, 49.5 ms after, **about +22 %**; the quietest run of each 33.6
against 41.5 ms. One pair at 1920 × 1080: 46.3 against 47.2 ms, which says the
cost is submission, not pixels. The fixed counts: draws 1 361 to 1 535, shadow
draws 2 840 to 3 115, triangles drawn 560 k to 603 k, static batches 1 187 to
1 586 -- almost all of them the district's windowed blocks. The probes cost
nothing measurable per frame (`--no-probes` against the default: 45.7/50.2
against 50.4/48.6 ms) and six to eight seconds at start-up. The machine was
not idle -- another agent was building CNA on it throughout, load average 5 to
11 -- which is why the table is eight pairs and not one number.

## 7. The frames that show it

- `pairs/09-car-three-metres.png` -- the black car, before and after. The
  single clearest picture of what the probes buy.
- `pairs/02-on-the-crossing.png` -- the white saloon's tail lamps and glass,
  and the far end of the street.
- `pairs/11-shop-window.png` -- the room behind the glass.
- `pairs/01-footway-looking-south-to-the-junction.png` -- the shopfront glass
  reflecting the street, the blue car's roof, the far end.
- `pairs/02-on-the-crossing-night.png` -- the sky.
- `after/15-kerbside.png` and `night/15-kerbside.png` -- the new photograph,
  day and night.

## 8. What still gives it away

1. **People.** At two metres a face is two eyes, a brow and a nose on a smooth
   head, and a garment is one colour. They read at street distance and not at
   arm's length. This is texture work -- a face and a garment atlas per
   figure -- and the atlas is also the largest remaining draw-call saving.
2. **Reflections are static and uncorrected for parallax.** A moving car is
   reflected in a shop window only through the sky term; a person not at all;
   and every surface reads its probe's cube as though it stood at the probe.
   Right in direction, approximate in position, and the approximation shows
   on the upper floors most.
3. **Trees.** Better greens and a viewpoint that shows them, but a crown is
   still cards, and from underneath the cards are visible as cards.

## 9. Repository state

Eleven code commits plus this documentation on `develop`, each building and
passing all ten test suites on its own. `validate-assets` clean; `git diff
--check` clean; `scripts/check-screenshots.sh` passes against the committed
set. No sibling repository touched; the exact `git status` of each is in the
closing report of the session and in section 0 above.
