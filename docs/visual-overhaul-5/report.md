# The fifth visual pass: coherence

A sixth pass over `cna-street`, with the narrowest brief yet. After four
passes the street had everything a street needs and most of it was good;
what gave it away was the *seams between* the good things. The authored
Astra in its bay was parked beside a white loft in the travel lane. The
authored people had a walk whose knees bent forward. The scanned brick and
render sat on planes with sixty identical holes in them. The composed cafe
had a box of packets next door. This pass added no rendering system. It
removed the discontinuities, one weakest-prominent-object at a time, and
took the two image-quality steps that were available on this side of the
framework.

Baselines, recorded before anything changed:

| Repository | Baseline | Note |
| --- | --- | --- |
| `cna-street` | `c171cec` on `develop` | The commit this pass starts from |
| `cna` (`../cnanext`) | `next` `1704c32` | Read-only throughout; another agent moved it to `22e89f1` during the session, and the final builds here picked that up |
| `sharp-runtime` (`../sharp-runtimenext`) | `next` `bfc826e` | Read-only |
| `easy-gl` | `develop` `deda7a4` | Read-only |
| `meta-gl` | `develop` `20c8b2d` | Read-only |

**No framework repository was modified.** Every change is in `cna-street`.
One CNA behaviour met on the way, GLTF-206, was worked around on this side
and the workaround is recorded under the finding in `docs/cna-findings.md`.

---

## 1. The strongest incoherences in the starting render

Rendered before changing anything -- all eighteen viewpoints at 1024 × 576
and the eight flagship frames at 1920 × 1080 -- and looked at, asking at
each one what the eye finds first that is fake.

1. **The moving traffic.** The car at three metres is an Astra and the car
   passing it is a white box with a black rectangle for a grille. The
   junction frame has three of them turning through it. One good authored
   car next to one loft is not a coherent frame; it is a frame that shows
   how a loft looks.
2. **The people's motion.** The figures are MakeHuman meshes and read as
   people at four metres, but every one of them walks the same walk, on the
   footway's exact centre line, in single file, snapping through right
   angles at the corners -- and, found when a row of eight was frozen
   mid-stride, with every knee bending forward.
3. **The upper floors.** The largest area in half the frames. Scanned
   render and brick on a plane, sixty windows of one size and one darkness
   with white frames, no keystone, no shutter, no curtain, nothing at the
   corner. The scanned surface made it worse: a real material on a
   generated pattern.
4. **The cafe and its neighbours.** The cafe's floor was a pale flat plane
   under a lit ceiling; the room beside it was a box of packets glowing
   through the glass, and the two are in the same footway frame.
5. **The pixels.** The Astra's tyre tread was a speckle, the leaf cards a
   shimmer, the shutter rails would have been a moiré: an imported image
   arrives with one mip level (GLTF-206), and the only anti-aliasing is
   FXAA over a single sample.

## 2. What changed

### The cars, driven (priority 1)

`scripts/blender-vehicles.py` now finds each authored model's four tyres and
splits the wheels off before it exports. Three ways, tried in order: by a
material named for a tyre (`gt_tire`, `Tire_MAT`, `Tread`, `Wheel`); by
connected pieces of the merged mesh that are cylinders of tyre size standing
on the ground; and, for a file whose mesh is a soup of loose triangles (the
Astra), by the objects that stand on the ground. From the tyre faces in each
quarter of the car it takes the axle as the centre of the tyre's own box --
not the ground and not the contact patch, which put the axle a few
centimetres off and made the wheel hop -- and everything whose bounds lie
inside that cylinder turns with the wheel: rim, hub cap, brake disc, but not
a caliper. Each wheel is exported as a node of its own, `wheel_fl` to
`wheel_rr`, centred on its axle with the axle as the node's translation; the
far level of detail keeps them welded on. Eight of eight models split; the
Mini turned out to face the other way from the seven others, which nobody
had seen while it was parked and everybody saw the first time it drove, and
is now turned.

`CityScene::buildHeroVehicles` loads each model as a whole (for the parked
copies, unchanged), as a body, and as four wheels with their axles read off
the node transforms, and deals the models to all thirty moving vehicles by
class -- the Sprinter where the simulation put a van, the Mini and the Punto
where it put a city car -- never the same model twice running on one lane.
The simulation is told each model's length, so the queue at the lights is
spaced for the car that is drawn, and carries a new `odometer` that never
wraps, so a wheel of any radius rolls from it without a seam. Per frame the
body is submitted with the vehicle's transform and each wheel with a roll
about its axle and, on the front pair, the steer angle. The lofts stay as the
fallback for a tree without the derived files, and as the simulation.

### The people (priority 2)

The walk is rebuilt from keyed gait curves rather than a sine: the thigh
forward to heel strike, back through stance to toe-off at six tenths of the
cycle; the knee nearly straight at heel strike, a little flex under load,
straight at mid-stance, sixty degrees at mid-swing; the ankle landing
toes-up and pushing off toes-down; the arms against the legs and trailing
them a little, the elbow bending most when the arm is forward, a hand's
breadth of abduction to keep the swinging arm off the hip; the pelvis
bobbing twice a cycle, swaying onto the stance leg, yawing the swinging hip
forward; the trunk counter-rotating and leaning in; the head turning back
against the trunk so it stays on the heading. The old walk had bent every
knee forward -- the sign was wrong -- which `gait_tests` now refuses.

Three walks and three stances: an ordinary pace, a brisk one with a longer
stride and more arm, an easy one with less; a wait that shifts its weight to
one leg and looks about, one that reads a phone with the head bowed to it,
one with the hands together in front. Every person is dealt a gait, a
stance, a stride scaled to their height and gait (the clip's clock is
distance over stride, so the feet stay on the ground in all three), a place
on the footway between the centre line and the buildings rather than the
line itself, and turns toward the way they are going over half a second
instead of snapping at a node; one person in six walks with the one in
front, on the same edge at the same pace a step behind and to the side, and
waits with them. All of it draws from a random stream of its own, so every
person stands where the fourth pass's seed put them. `--lineup` now stands
eight people and freezes eight more at successive eighths of the walk, which
is how the knee was found.

### The facades (priority 3)

The windows on the older blocks are two metres tall, as the rooms behind
them were built for; at 1.65 m the upper floors were two thirds wall.
Behind every window is something: a net curtain across the pane or drawn to
the sides, a roller blind part way down with its bar, or a room with the
light off, each a different tone and a different depth in the opening. The
masonry blocks get a flat hood or a moulded architrave over every head and a
keystone standing proud of it; half the rendered blocks carry folding
shutters beside their windows in the plot's door colour, panelled with three
rails and an arris, one window in eight closed; half the masonry blocks have
alternating quoins up both corners. The dressing draws from a stream of its
own per elevation, so adding a curtain to a window did not re-deal the
balconies and the brick colour of every plot after it -- the first version
did exactly that, and the before/after pairs would have compared two
different streets.

### The cafe, and the shops beside it (priority 4)

The cafe: an oak floor a stop darker than the tables on it, three beams
across the ceiling, a dado rail and boarded panelling to hip height along
both side walls with a skirting under, a boarded front to the counter over a
kick recess with the top overhanging it, timber shelving across the back
wall with bread, baskets and croissants on it in place of the white steel
rack that glowed like a fridge, a shelf of tea things along the left wall
at head height and a bench under the prints; outside, bronze joinery instead
of the street's white, and a hanging sign on a wrought bracket at the corner
of the fascia so the cafe reads from along the footway.

Every other shop now asks the scene for the scans its trade would have --
crates and cartons and a sack truck for the grocer, baskets and a chalkboard
for a bakery, planters and plants for the florist, a print and a clock for
the office -- one hundred and fifty-four stood over the street. The stock on
the generic shelves is a stop darker and a third less saturated, and the
ceiling tubes a third dimmer: through glass in daylight the full-colour grid
of packets under blooming tubes was the loudest surface on the street. The
potted plant, a 176 000-triangle scan, is kept to the florist.

### Vegetation (priority 5)

The hero corridor's trees were already the three scanned species, and every
flagship frame's foliage is a scan; nothing here needed replacing, and the
litter bin -- the one grey generated cylinder among scanned hydrants,
benches and cafe furniture in the footway frame -- is now the scanned steel
can. The generated trees stand beyond eighty metres, where they are a few
pixels of green.

### Image stability (priority 6)

Two things, both on this side of the framework.

**Mip chains for every imported image.** `ContentManager` looks for
`<name>.cnb` before it looks for a loose `<name>`, and a compiled model
refers to its images by full file name. So the content build now compiles
every image a model refers to under that full name --
`car-honda-civic-ek_tex0.png.cnb` -- through the same mip-chain compiler the
catalogue's surfaces go through, in the colour space
`scripts/model-textures.py` reads off the model's descriptor: sRGB for a base
colour, linear for a normal or a metallic-roughness map. The model loads as
before and its textures arrive with a chain; `ModelLibrary` logs any that
do not, and none do. Every scanned prop, tree and car stops shimmering at
distance, and the Astra and the Civic, parked nearest the close viewpoints,
carry 2k paint that holds at three metres. GLTF-206 stands as a framework
finding; this is the workaround.

**Supersampled stills.** `--supersample 2` renders a still into a back
buffer twice the size and box-filters it down in linear light on the way to
the file. It is the one anti-aliasing this application can add without a
change in CNA, and the only one that settles tyre tread, wheel spokes, leaf
cards and shutter rails smaller than a pixel. It costs a quadrupled frame and
about 4 GB of memory, so it is for stills: the flagship frames are shot with
it, the before/after pairs are not, and the window never is.

### A scan that drew black

Standing the scanned litter bin in the footway frame showed it as a black
silhouette, and the reason turned out to be older than the bin: thirteen
of the imported models -- the street bench, the condenser units, the
ceiling lamps, the fluorescent fittings, the bin, three of the Khronos
props -- ship a packed occlusion-roughness-metalness image whose occlusion
channel is empty, and `PbrEffect` multiplies the lighting by it. The bench
and the condensers had been a stop too dark since the third pass.
`ModelLibrary` now reads a patch from the middle of every packed map once,
and where the channel is blank sets the material's occlusion strength to
zero and says so in the log. Not a CNA matter: the images are what they
are, and an importer cannot know a zero is a mistake.

### Connection detail (priority 7)

The hanging sign's bracket, the shutters' rails, the counter's kick, the
bench and the shelf brackets, the scanned bin standing on its base at a
street bin's height: small, and each one a thing that meets another thing
the way it does in a street.

## 3. Tried and rejected

- **Splitting the wheels by connected pieces alone.** The first version of
  the finder separated every model into loose pieces; the Astra came out as
  twenty thousand triangles and no tyre, because its export doubles every
  vertex along every seam. Merging doubles first, then classifying faces
  rather than objects, then falling back to the names of the nodes that
  stand on the ground, is what got all eight.
- **The axle from the ground.** Taking the axle as half the tyre's height
  above z = 0 and its centre from the contact patch put it off the tyre's
  centre on models a few centimetres off level, and the wheel hopped. The
  tyre's own box is the axle.
- **Re-dealing the ornament from the plot stream.** The first window
  dressing drew from the plot's random stream and shifted every decision
  after it: balconies appeared, a brick block turned red. A stream per
  elevation for the dressing alone.
- **A potted plant in every shop window.** 176 000 triangles a copy; a
  dozen copies was three million triangles for a plant. The florist keeps
  two.
- **Supersampling for the regression screenshots.** A 4 GB, eighty-second
  capture per viewpoint would make `check-screenshots.sh` a test somebody
  turns off. The committed set and the pairs stay at one sample; the
  flagship set is resolved at two.
- **The scanned bin as shipped.** The file holds two cans side by side, a
  clean one and a rusted one, and the first placement stood both at every
  bin; the clean one's four nodes are taken.
- **Tinting the authored cars' paint** for variety among the thirty moving
  copies. The textures are baked with their colour; a tint over a red Mini
  is a dark Mini. Left for a model whose paint is neutral enough to take
  one.

## 4. CNA capabilities used

Nothing new was called. What was used more fully: `ContentManager`'s asset
precedence, which puts a compiled `.cnb` in front of a loose image and is
what the mip-chain workaround stands on; `Texture2D::CreateFromPixels` and
`GetBackBufferData` at four times the frame; `AnimationPlayer` on six named
clips per skin instead of two; the importer's node-to-`ModelMesh` naming
(`wheel_fl_0`, `wheel_fl_1`, one per primitive), which is how a wheel is
picked out of a file.

## 5. Performance

See `performance.md`.

## 6. The frames that show it

- `pairs/09-car-three-metres.png` -- the loft beside the Astra becomes an
  authored car, and the traffic behind it too.
- `pairs/16-corner-to-corner.png` -- the junction: three lofts become a VAZ,
  a Punto and a Logan; the corner block gets quoins, shutters and keystones.
- `pairs/10-pedestrian-four-metres.png` -- a walk with the knees bending the
  right way, an arm swinging, a companion.
- `pairs/11-shop-window.png` -- the cafe with a floor, beams and a counter
  rather than a pale plane.
- `pairs/05-looking-up-at-the-facades.png` -- the upper floors dressed.
- `flagship/` -- at 1920 × 1080, at one sample and at two.

## 7. What still gives it away

1. **The people's faces at two metres**, unchanged: a painted skin is a
   painted skin at conversational distance. The motion now holds at four
   metres, which is where the flagship frames stand.
2. **The far end of the street.** Past eighty metres the trees are the
   generated ones and the facades the fourth pass's, and a long view down
   the carriageway shows where the hero corridor stops.
3. **The draw count.** On the machine's own integrated GPU the frame is
   bound by submission -- about twenty microseconds a draw through the
   OpenGL 3.3 path and three and a half thousand shadow draws, one per
   instance (CNA-F6) -- so a street of 1 780 draws runs at eleven frames a
   second with everything on and twenty-one with the shadows off; see
   `performance.md`. The people are three hundred of those draws and a
   moving car up to thirty-three. Fewer draws is this project's to do;
   the cost of a draw is the framework's, and this pass changed nothing
   there.
