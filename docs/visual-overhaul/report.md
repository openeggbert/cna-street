# The visual overhaul: what was done, what it cost, and what is left

A second pass over `cna-street` with the *rendered result* as the acceptance
criterion rather than the code. The engineering was already sound; the pictures
were not, and this is the record of closing that gap.

Baselines, recorded before anything changed:

| Repository | Baseline | Note |
| --- | --- | --- |
| `cna-street` | `c4f80ce` | The commit this overhaul starts from |
| `cna` | `3471395` (upstream `next` at `756096`) | Unchanged by this work |
| `sharp-runtime` | `bd282d1` | Unchanged |
| `easy-gl` | `deda7a4` | Unchanged |
| `meta-gl` | `2520173` | Unchanged |

**No framework repository was modified.** Everything below is in `cna-street`.
The CNA defects found are recorded in `docs/cna-findings.md` with reproductions;
the one CNA change this project ever needed (CNA-F12, the content pipeline's
missing mip chain) predates this overhaul and is already on a branch with a
patch in `docs/patches/`.

---

## 1. What was actually wrong

The audit rendered every viewpoint and listed twelve failures. They fell into
three groups, and the grouping turned out to matter more than the list.

**Things that were bugs.** The sun was blue. Its colour was being taken from the
sky model's radiance a fraction of a degree off the disc, which is *scattered*
light and therefore goes as 1/λ⁴ — so the street was lit by a blue sun under a
blue sky and no exposure could put warmth into a frame that never had any. The
environment cube was Reinhard-tone-mapped and sRGB-encoded on the way in and
read back as linear. And every roughness in the catalogue was its own square,
which is section 3 below.

**Things that were the wrong shape.** Vehicles assembled from boxes. People with
no neck, a cylinder for a torso and a sphere for a head. Trees that were a
cylinder with four straight limbs. These read as programmer art at any distance
and no amount of material work would have saved them.

**Things that were missing.** Nothing behind the shop glass. No night. No
viewpoint closer than ten metres, which is why several of the above survived
eight rounds of screenshots.

---

## 2. What changed

### Lighting

Sun colour is now atmospheric *transmittance* along the solar path — Rayleigh
coefficients at 680/550/440 nm against Kasten–Young air mass, which does not
diverge at the horizon the way 1/sin does, with Mie carrying the turbidity.
Sunlight is what survives the scattering that makes the sky blue, so it is the
complement: warm, and warmer the lower the sun.

The environment cube is a plain linear quantiser with its scale carried on
`ImageBasedLightEXT::Intensity`, which is the field documented for exactly this
— the products are 8-bit and the radiance is not.

This was the single largest visual change in the whole overhaul and it is two
functions.

### Materials — the PBR factor bug

`PbrEffect` computes `roughness = map.g * roughnessFactor` and
`metalness = map.b * metallicFactor`. This catalogue was violating that on every
one of sixty-odd surfaces, in the most plausible way there is: the generator is
handed the surface's roughness, writes it into the map with its own variation
around it, and then the material declares *the same number* as the factor.

```
painted        R: 0.380 x 0.422 = 0.160        M: 0.00 x 0.00 = 0.000
sign-street    R: 0.300 x 0.372 = 0.112        M: 0.00 x 0.00 = 0.000
wheel-track    R: 0.580 x 0.670 = 0.389        M: 0.00 x 0.00 = 0.000
car-body       R: 0.160 x 0.086 = 0.014        M: 0.00 x 0.47 = 0.000
```

Painted metal asked for 0.38 and got a gloss lacquer, so every lamp post,
bollard, bin, sign back and window frame in the city was lacquered. A road sign
asked for 0.32 and got 0.12. A wheel track asked for 0.58 and got 0.39, which is
why the carriageway looked wet from a low camera. Metalness failed the other way
and silently: a material declaring itself metal over a map whose blue channel is
zero is a dielectric however emphatic the declaration, so every galvanised post
and alloy wheel in the scene was plastic.

The factors are now divided by what the generator actually wrote, so the product
*averages* the declaration and keeps the map's spatial detail in proportion. The
divisors are measured at bake time and travel with the compiled content in
`surfaces.txt`, because the content path never runs a generator and a fix that
only works when the textures are generated is not a fix.

Nothing about this looks wrong in a diff. It took printing both columns.

### Geometry: a parametric surface primitive

`SurfacePatch` sweeps a lattice and derives *vertex* normals from neighbouring
faces — but only across edges whose faces agree within a smoothing angle. That
threshold is what keeps a car's shoulder crease, its wheel-arch lip and its
bonnet shut sharp while the panels between them stay smooth. Averaging
everything gives a melted car; averaging nothing gives the faceted one. Every
vehicle body, every human limb and every tree trunk in the scene goes through it.

### Vehicles

Twelve variants over six classes — city car, hatchback, saloon, estate,
crossover, van — each described by five monotone-cubic curves (roof, belt,
rocker, half-width, upper half) rather than assembled from boxes. Wheel arches
are *cut into* the body by an ellipse wider than it is tall. Wheels are surfaces
of revolution with shoulders, a brake disc, a five-spoke rim; they roll, steer,
and the shut lines, handles and mirrors sit on the flank because they are placed
by the same function that generates it. Brake lamps light, and vehicles turn
through the junction on a quadratic Bézier.

### People

One mesh per person on a nineteen-bone skeleton, weighted by distance, animated
by `AnimationPlayer` into `SkinnedPbrEffect` from walk and idle clips built in
code. The clock is the distance walked rather than wall-clock time, so nobody's
feet slide and nobody's queue breathes in unison.

Two rigging bugs are worth recording because both look like rendering faults:

- A keyframe replaces a bone's *whole* local transform, so a track that leaves
  the translation at zero collapses its bone onto its parent. The first figures
  had no legs and one arm growing out of the neck.
- A bone catches the flesh of the limbs running *out* of it to its children, not
  the limb running into it from its parent. Measuring the latter is defensible
  on paper and puts the upper arm's flesh on the forearm bone, so every limb
  bends about the wrong joint.

And then, with the rig correct and the clips playing, the figures *still* read
as mannequins — and all three causes were geometry, found by standing one at two
metres and looking at it: an arm whose first section sat above the shoulder line
beside the neck (a wing); a hairline ring drawn low enough to cross the eyes (a
welding mask); hand sections eight centimetres across and ten deep (a boxing
glove). None of those is visible in a diff.

### Shops

Every ground-floor unit is a room. What it sells is decided once per plot, so
the lettering on the fascia and the fittings behind the glass cannot disagree,
and some are vacant with a to-let notice. Shelving with stock, a counter with a
till, display plinths, lit ceiling strips.

Two lighting problems had to be solved to make them read. Having correctly
decided that interiors need no shadows and no room-scale occlusion, the rooms
were lit as though their walls were not there — full sky ambient, full IBL, the
same brightness as the pavement. Occlusion in the ORM map's red channel is the
right lever, because `PbrEffect` multiplies both ambient terms by it and leaves
direct light alone. That fixed the brightness and exposed the next thing: what
little sky a shop sees is *blue*, so every packet came out navy. A real shop is
lit by warm fluorescent tubes; the ceiling strips here glow but emit nothing and
`PbrEffect` carries one punctual light per draw, not thirty-nine. So the light
is baked — each interior surface takes an emissive map that is a copy of its own
albedo times a warm factor, which is exactly "warm light bouncing off this
colour", so a red packet stays red.

### Surfaces

The carriageway read as gravel because it could not read as anything else. At
512 over a six-metre tile it had 1.2 cm per texel, so the 8–16 mm aggregate that
asphalt *is* was unresolvable whatever the generator did. The road gets its own
1024 map and a five-metre tile: 0.49 cm per texel, a 3 cm chip six texels across.
The chips are flush rather than domed (they are rolled in), the crack field is
thresholded near its crest instead of covering a quarter of the tile, and the
oil is a third of what it was.

The footway had the right slab size and the right per-slab variation and still
read as a drawn grid, because the tile held *nine* slabs and repeated every
1.5 m. What the eye picks up is the group. Sixty-four slabs on a four-metre tile,
plus lippage, a hairline crack on one slab in twenty and the grit along a walking
line.

### Vegetation

Trunks are swept surfaces with a root flare and a constant radial wobble.
Branching is recursive to five levels, and `spread` — which was declared in the
species table and never read, so every tree came out the same narrow shape — now
sets the first limb's length. Foliage hangs at three points along every twig and
on the two levels above it, because density in a canopy is the number of
*separate* places foliage hangs from: three cards through one point cover one
card's worth of sky.

The last part is lighting. Each leaf card's vertex normals are bent most of the
way from the card's own facing toward the direction out of the crown's centre,
flattened vertically because a crown is wider than it is deep. The canopy shades
like a ball for four normalisations per card, and that is the difference between
foliage and bunting.

### External assets

Sixteen glTF models, fetched not committed, each with its licence established
individually. The manifest records name, title, author, copyright, source URL and
repository, exact licence and licence URL, whether attribution is required,
whether redistribution is allowed, retrieval date, original format, SHA-256, byte
count, transformations and role. All sixteen are CC0-1.0 or CC-BY-4.0; three
candidates were rejected and the manifest says why (Duck is SCEA, Sponza is a
CRYENGINE agreement, Virtual City is testing-only).

`scripts/validate-assets.py` checks the mechanical half and found a problem on
its first run: a 7.8 MB model sitting in the fetched set, reaching the content
build, with no manifest entry and therefore no established licence.

**A network constraint shaped this.** The egress proxy here refuses Poly Haven,
ambientCG, Kenney, `github.com` HTML and `codeload`; `raw.githubusercontent.com`
is what it allows. The set is therefore a materials showcase rather than street
furniture, which is why the props behind the glass are a lantern, a boombox and
a vase of flowers rather than a mannequin and a shelf of shoes.

### Night

`--night` puts the sun four degrees under the horizon and the street lights
itself: luminaires, the pools they throw on the road, shop windows, the flats
above them, headlights and tail lights. It costs 3% of the frame, because
everything but the pools is an emissive material on geometry that was already
being drawn.

---

## 3. Performance

Measured with `--frames N`, which discards six warm-up frames and prints mean,
median, p95, min and max. Run-to-run variation here is ±60 ms, so every
comparison was measured by interleaving the two builds in one session.

| | baseline `c4f80ce` | current |
|---|---:|---:|
| median frame, 1024 × 576 | 413–434 ms | 467–506 ms |
| draw calls | 1 159 | 1 356 |
| shadow draw calls | 2 269 | 2 819 |
| triangles drawn per frame | 531 045 | 559 349 |

**+15%**, and the interesting part is how it got there. The intermediate state
was +26%, with drawn triangles up 11% — which said immediately that this was not
a triangle problem. Of 1 997 draws, 351 were skinned characters and about 500
were wheels, a wheel being three materials at four corners.

| change | draws |
|---|---|
| Wheels baked into the distant body | ~500 → 0 |
| Character material set collapsed past 20 m (6 → 3) | 351 → 228 |
| Pedestrian cull distance of its own (210 m → 130 m) | 228 → 160 |
| Distant vehicle: trim folded into underbody, plate dropped | 46 → 0 |

Nothing there changes a silhouette. Wheels stop rotating past 38 m where a wheel
is eight pixels across; a figure past 20 m loses the material boundary between
its shoes and its trousers, at which point it is fifty pixels tall.

At 1920 × 1080 the frame is 1 031 ms, of which the post chain is 694 — six
full-screen passes at 2.07 megapixels on four CPU cores. That line would be the
smallest in the table on any hardware with a rasteriser in it.

---

## 4. What was tried and dropped

**Screen-space reflections.** Wired to a setting and off by default. On the shop
window they wash the interior out and cannot reach the glass at all — it is
alpha-blended and drawn after the pass. On the road and the cars they change
nothing visible, because the asphalt is far rougher than any SSR cutoff and a
black car reflecting dark asphalt has nothing to show. The pass works; this
scene has no surface for it.

**A per-material shadow-distance cap.** Implemented, measured, kept, and its
uselessness written into the code. One plot's window frames are a *single* batch
spanning a whole elevation, and a distance test against a batch takes its nearest
corner, so an elevation running from 20 m to 60 m is kept whole. Per-batch
culling can only be as fine as the batches, and these are coarse on purpose.

**The imported rig on the pavement.** See section 6.

---

## 5. CNA findings

Two new defects and one behaviour note, all in `docs/cna-findings.md` with
reproductions.

**CNA-F14 — a skinned figure cannot cast its own shadow.** The cascade caster
takes its world matrix from a uniform and knows nothing about a bone palette.
Every character here carries a rigid stand-in in its bind pose, submitted
shadow-only. Proposed fix: a skinned variant of the caster program taking the
same palette `SkinnedPbrEffect` already takes.

**GLTF-206 — imported glTF textures arrive without a mip chain.** The
application's own surfaces get one; a model's referenced images do not. A 4K
texture on a 40 cm prop aliases whenever the camera moves.

**GLTF-207 — a model's skin is on `SkinsEXT` or on `Tag`, depending on how it
was loaded.** The runtime glTF path fills `Model::SkinsEXT`; the compiled `.cnb`
path leaves it empty and puts the `SkinningData` on `Model::Tag`. `SkinsEXT` is
the better API and the one the header recommends, so a caller written against it
gets an empty vector from every model that went through the content pipeline,
silently — and the compiled path is the one a shipping application uses.

---

## 6. What is unresolved

**GLTF-208 — an imported skinned mesh part draws nothing.** Recorded as
unresolved rather than worked around or quietly dropped.

Everything that can be checked, checks out: the model loads (4 672 triangles,
1.531 m as authored); the skeleton loads (19 bones, hierarchy, bind pose, inverse
bind pose, one named clip); `AnimationPlayer` produces 19 well-formed matrices;
the material is opaque with its texture bound; the transform puts the figure
exactly where it was asked to stand; and **the vertex declaration matches the
effect's byte for byte** — stride 68, same six elements at the same offsets in
the same formats. Drawing the same part *rigidly* also renders nothing, while
every unskinned imported model in the same scene renders correctly through that
path, so it is neither the skinning nor the placement.

Ruled out: winding, the culling volume (which was a real bug in the caller and is
fixed), the vertex layout, the palette, the material, the transform. Not ruled
out: the index element size (the compiled part uses 16-bit indices where every
generated mesh here uses 32-bit) and the vertex/index offsets for a part that is
not the first in its buffer.

`ModelLibrary::loadRig` is kept and called at start-up, so the round trip — glTF
to `.cnb` to `Model` to `SkinningData` to `AnimationPlayer` — is exercised and
logged on every run. The figure is *not* placed in the crowd, and the manifest no
longer claims it walks the street. An invisible pedestrian plus a line in the
documentation saying otherwise would be worse than a missing feature.

---

## 7. Testing

Nine suites, all passing in Debug and Release. `appearance_tests` is new: nine
cases, each one a defect that was shipped, found by looking at a rendering, and
fixed — a generator writes the roughness it was handed; painted metal carries
unit metalness so a material's factor can mean something; a per-cell hash reaches
both ends of its range where value noise at a half-integer lattice point never
leaves the middle; four corners in the obvious order wind clockwise seen from
above; sixty-four slabs give a spread of tones where nine gave a repeating block;
asphalt's relief belongs in the thousandths.

Not one of them would have been caught by a pixel comparison, which is the point:
a pixel test tells you something changed, and these tell you what a surface is.

---

## 8. Repository state

Twenty commits on `claude/cna-city-street-demo-ut9lp1`, each one coherent.
Clean Debug and Release builds with no warnings in this project's code. Content
deleted and rebuilt from source (202 surfaces, 11 models). All nine test suites
pass in both configurations. `validate-assets` clean. `git diff --check` clean.

---

## 9. The question

> Would a stranger, shown a still from this, say *"a tech demo with programmer
> assets"* or *"a surprisingly convincing rendered city street"*?

Honestly: the second, for most of the fourteen viewpoints, and not yet for all
of them.

What carries it: the street reads as a *place* — the proportions are real, the
buildings have depth, the surfaces have the right scale, the light is warm and
directional, the shop windows are lit from inside, the cars are cars and the
people are people. The night set is the strongest work here.

What still gives it away, in order: car paint has no environment to reflect but
the sky, so a dark car is a dark shape rather than a dark *car*; shop glass has
no reflection at all, which is the single most missed cue at eye level; and the
distant context is massing with a printed façade. Those are the three things a
next pass should take, and the first two are the same problem — this renderer has
no planar or screen-space reflection that works on the surfaces that need one.
