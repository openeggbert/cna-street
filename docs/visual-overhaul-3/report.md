# The third visual pass: content before systems

A fourth pass over `cna-street`, with a different brief from the three before
it. The renderer already had shadows, an HDR chain, image-based lighting,
reflection probes and a glTF material model, and the street still looked like
a technically impressive procedural demo rather than an environment-art
screenshot. The ceiling had moved from the renderer to the content, so this
pass spent itself on content: scanned surfaces, scanned props, a scanned tree,
and then the light those surfaces are seen by.

Baselines, recorded before anything changed:

| Repository | Baseline | Note |
| --- | --- | --- |
| `cna-street` | `83dc8e1` on `develop` | The commit this pass starts from |
| `cna` (`../cnanext`) | `next` `e1d3aa5` | Read-only throughout |
| `sharp-runtime` (`../sharp-runtimenext`) | `next` `c3fbb95` | Read-only |
| `easy-gl` | `develop` `deda7a4` | Read-only |
| `meta-gl` | `develop` `20c8b2d` | Read-only |

**No framework repository was modified.** Every change is in `cna-street`.
The CNA behaviour met along the way is in `docs/cna-findings.md` under *Third
visual pass*; none of it is a defect that needed working around.

---

## 1. What was wrong with the starting render

Rendered before changing anything, all sixteen viewpoints, and looked at
rather than reasoned about. In order of visible damage:

1. **Every surface said "procedural".** The render was a noise field with
   speckles, the footway a grid with a crack texture on it, the asphalt dark
   grey with weather, the brick a pattern. Each generator had been pushed as
   far as a description of a material goes in code, and what remained was the
   gap between a description and a photograph.
2. **The frame was a stop and a half underexposed and the sky was brighter
   than a sunlit wall.** A sunlit render sat at 35 % on the display; the shade
   was black; the horizon sky radiance was nearly half the sun's, which is a
   sky three to four times too bright for its sun, and the exposure had been
   set low to keep it from clipping.
3. **The street furniture was primitives.** A hydrant was a red cylinder with
   a ball on it, a cabinet a grey box, a bench planks on a frame. Furniture
   is the scale reference the eye uses, and it is also the thing the eye
   inspects first.
4. **The trees were leaf cards on a swept trunk.** Better than the lollipops
   of the first overhaul and still bunting from underneath.
5. **The imported shop props were untextured.** This was not known at the
   start; it came out when the scanned props also arrived white. The material
   registry had been discarding every imported model's texture pointers since
   the first overhaul, and a white vase of flowers looks like a vase of
   flowers.
6. **Vehicles still read as lofts** -- better lofts than before, and lofts.
7. **People still read as mannequins.**

## 2. What changed

### Scanned surfaces

Fourteen photogrammetry PBR sets from Poly Haven, all CC0, replace the
generated asphalt (main, side and worn), the paving slabs, the granite kerb,
the red, buff and engineering brick, the lime render, the sandstone ashlar,
the concrete panels, the roof tiles, the plane-tree bark and the shop floor.

They enter through the content build, not the catalogue.
`scripts/prepare-surfaces.py` writes each scan's three maps over the generated
ones under the generated surface's name and writes `authored.txt` beside them;
`MaterialLibrary` reads that file and, for a scanned surface, takes the
roughness and metalness maps at face value, applies the UV scale that maps the
geometry's tile onto the scan's physical size, and resets the tint unless the
scan was neutralised for tinting. The render *is* neutralised -- divided by
its mean colour in linear light -- so the catalogue's seven facade tints still
cost one texture. A tree that has not fetched the scans, or has no Pillow,
gets the generated surfaces and a message.

One convention had to be found the hard way. The scans arrive in the OpenGL
normal-map convention; this project's meshes carry a tangent frame whose
bitangent runs along increasing v, which is the DirectX convention, and the
generator has always written its maps to match. A scan laid on those meshes
rendered inside out, and a synthetic map of hemispherical bumps under a low
sun -- which came out as bowls -- settled it. The green channel is inverted
in the preparation step and the reason is written where the flag is.

### Scanned props

Twenty Poly Haven models through CNA's own glTF importer, unchanged files,
each placed where the generated prop it replaces was placed and each with that
generated prop kept as the fallback: the fire hydrant at the kerb, two utility
cabinets against the building line, the street bench (one module of a
modular kit, taken by node name), planter boxes with two shrub clumps sunk in
the soil, manhole covers on the crown of the road where the road builder
painted them, cafe tables and chairs and an A-board on the footway outside
each bakery, crates and cartons beside one shop door in five, a refuse sack
beside one bin in three, and a car under a cover in the first free bay of
the west parking lane between the shop-window and kerbside viewpoints.

Two things had to be added to make that possible. A prop's part now carries a
local transform, so a model whose file ships a fresh hydrant at x = -0.3 and
an aged one at +0.3 yields one hydrant standing on the origin, and a shrub can
be appended into a planter. And `MaterialLibrary::add` no longer overwrites
the textures the importer bound with the null an empty upload returns, which
is the one-condition fix that also gave the Khronos shop props their textures
back.

### Hero trees

A scanned small tree (Poly Haven `tree_small_02`), cut in Blender by
`scripts/blender-tree-lod.py` from its half-million-triangle LOD1 to a
158 000-triangle street level of detail and a 38 000-triangle far one -- the
wood decimated, 40 % and 10 % of the leaf clusters kept and scaled up 1.4x and
2.6x so the crown keeps its density and silhouette -- with the leaves rewritten
from Blender's BLEND to MASK so they write depth and cast shadows. It stands at
every tree pit on the west footway between the junction and 68 m north of it,
scaled 1.55x to 1.85x to a street tree's height, with the far copy past 55 m;
the generated trees stand everywhere else. The leaf cards' specular is cut to
a third and their roughness raised at import, because a card seen from both
sides with one normal otherwise mirrors the sky from underneath.

### Light

Sun 4.2, sky 0.85, exposure 0.9 in place of 3.0, 1.0 and 0.42. A sunlit render
now sits at about 85 % on the display, the zenith at about 55 %, the horizon
haze near white, and the shade a little under three stops down. The reflection
probes' irradiance is convolved from the capture multiplied by 1.6
(`probeBounceGain`), because a capture holds one bounce and a canyon of light
render is lit by several; the specular keeps the plain capture. The sky's own
environment gained the sunlit ground under it. `--night` is unchanged at its
own exposure and was checked.

### The shop interiors

A scanned terrazzo floor, with the occlusion scaled and the room-light emissive
copy written by the preparation step exactly as the catalogue does for every
surface inside a shop; the packaging on the shelves half as saturated and more
of it plain card; and the Khronos props on the plinths textured for the first
time.

### The record

The manifest grew a `surfaces` list, multi-file assets, derived files and a
`compile` flag; `scripts/manifest-tool.py` is now the one reader of its shape,
used by the configure step and the fetch script; the fetch script fetches
every declared file; the validator checks surfaces, multi-file assets and
derived files and refuses undeclared files anywhere under `downloads/`, and is
registered with CTest. `assets/ATTRIBUTION.md` is generated from the manifest
by `scripts/attribution-table.py`.

## 3. CNA capabilities used

Nothing new was called to be listed. What was used more fully:

- `cna_tool_gltf_to_cnb` on `.gltf` documents with external `.bin` and
  `textures/` folders, twenty times, and on Blender-exported `.glb` files with
  alpha-masked materials.
- `PbrEffect`'s texture pointers off an imported part, now actually kept.
- `TextureTransformEXT` for the physical UV scale of every scanned surface,
  which was the mechanism the catalogue already used for atlas cells.
- `EnvironmentProcessor::generateIrradiance` on a second cube per probe, the
  capture at the bounce gain, alongside the specular from the first.
- `AlphaModeEXT::Mask` with `doubleSided` for a hundred thousand scanned leaf
  cards per hero tree, which the shadow pass and the prepass both honour.

## 4. Tried and rejected

- **Sky irradiance in place of the probes' (setting `probeIrradiance` off).**
  Lifted the shade to where it belonged and lost the canyon: every wall the
  same brightness whichever way it faced. The bounce gain gives the same lift
  and keeps the probes.
- **Uncharted 2 in place of the filmic curve.** Brighter mids, a whiter sky
  and less separation between sun and shade at the same exposure; the filmic
  curve at 0.9 kept more of the picture.
- **Blended leaves.** What Blender exported: the crown drawn in the
  transparent phase without depth, a ghost against the sky. Masked instead.
- **A per-island delete in the tree script.** Correct and quadratic; the
  first run had not finished after twenty-five minutes. One delete for the
  whole mesh finishes in under a minute.
- **The concrete pavement and painted plaster scans** as alternatives to the
  square slabs and the plastered wall: fetched, compared, not used.
- **A second scanned tree species** (Poly Haven's island trees): fetched and
  dropped for time; one species scaled with variation reads as a planted row.

## 5. Performance

See `performance.md`. Measured with `--frames 26` at viewpoint 1, the two
builds interleaved in one session. The short version: draw calls are within
three per cent of the baseline, triangles drawn per frame roughly triple
(the hero trees), and the frame time on this software rasteriser moved by
under ten per cent at both 1024 × 576 and 1920 × 1080, because the frame here
is bound by submission rather than by pixels or triangles. Scene build gained
the scanned models' load; the probe bake is unchanged.

## 6. The frames that show it

- `pairs/01-footway-looking-south-to-the-junction.png` -- the whole change in
  one frame: paving, render, hydrant, cafe, tree, light.
- `pairs/09-car-three-metres.png` -- the asphalt and the light on the car.
- `pairs/13-street-tree.png` -- a scanned tree where a card tree stood.
- `pairs/15-kerbside.png` -- the slabs, the kerb, the bark.
- `pairs/16-corner-to-corner.png` -- the ashlar corner block.
- `after/17-pavement-cafe.png` and `after/18-covered-car.png` -- the two new
  photographs; no before exists.
- `flagship/` -- seven of them at 1920 × 1080.

## 7. What still gives it away

1. **The vehicles.** Every car but the covered one is a loft, and from three
   metres a loft is a loft: no shut lines, no panel curvature, glass that is
   a tinted plane. The one scanned vehicle shows exactly how far the others
   have to go.
2. **The people.** Mannequins at four metres, and the photographs are framed
   to keep them out of the foreground.
3. **The shop interiors.** Real floors and textured props now, and still a
   shelf of packets behind a pale plinth in a box lit from a ceiling of tubes.
   Scanned shelving and stock, darker and deeper rooms, and a door that reads
   as a door are the next pass.
