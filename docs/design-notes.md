# Design notes

The decisions behind the code, and what happened when they were made
differently. The README says what the project is; this says why it is shaped
the way it is.

## Proportion is the whole game

`street/include/CnaStreet/Scene/StreetMetrics.hpp` is the most important file
in the project. A street reads as fake long before anyone can say why, and the
reason is almost always proportion: a sidewalk you could park on, a car the size
of a bus, a doorway a giant would use. Nobody looks at a render and thinks "that
kerb is 40 cm"; they think "that looks like a game".

So every dimension is a named constant in one header with a note saying where it
comes from — German RASt 06 and EAE street-design practice for the highway, DIN
18065 for storey heights, StVO sign sizes, DIN 67523 for lighting geometry — and
every generator reads them. The lane is 3.30 m because that is what an urban
main road with buses and no trams is built to. The kerb is 14 cm. The lamp is
8.00 m with a 1.55 m outreach. A bollard is 90 cm, and a bollard 30 cm too tall
makes the whole street feel small in a way nobody can put their finger on.

Street furniture matters most of all, because it is the scale reference a viewer
actually uses. A building could be any size; a bench could not.

## Everything is generated, and that is architectural

There is no downloaded texture, model, font or sound anywhere in this
repository. That is not a way of avoiding a licence question — it is a set of
properties worth having:

* A generated surface can be produced at any resolution.
* It tiles by construction, rather than because someone made it tile.
* Its roughness map is derived from the same height field as its normal map, so
  the two agree, rather than being guessed at separately.
* It comes from a seed, so it is byte-identical on every machine that builds the
  repository — which is what makes a golden screenshot mean anything.
* The whole asset set is reproducible from source with one command and no
  network.

The cost is that each surface has to be *authored*. `TextureFactory.cpp` is a
description of what asphalt looks like — 8–16 mm aggregate in a bitumen binder
that greys as the binder wears off the stone, polished wheel paths, cut-and-fill
patches over a service trench with a sealed lip, branching fatigue cracks, oil —
not a call into a noise library. That is a lot of code for a road surface, and
it is the difference between a road and a grey plane.

## The scale a texture is drawn at is a property of the material

The single most instructive bug in the project: the asphalt drew its aggregate
as 46 cells across the tile, and the road lays that tile at six metres, so every
stone was 13 cm across. From the footway the carriageway looked like bubble
wrap.

A 6 m tile in a 512 px map is 1.2 cm per texel. The largest aggregate that can
be *resolved at all* is about 4 cm, and drawing anything finer is drawing noise.
The fix was not a better noise function; it was doing the arithmetic.

The same arithmetic says where a feature cannot live in a material at all. Wheel
tracks are a property of a *lane* — of which way the traffic runs — and a tiling
texture has no idea which way that is. The version that drew them in the
material put them across the carriageway, repeating every three metres, which is
a road surface no vehicle has ever made. They are decals along each lane now.

## Winding, and why it deserves its own function

CNA follows XNA: with the default `RasterizerState::CullCounterClockwise`, a
face is visible when its vertices are clockwise as seen from the front. That
convention lives in exactly one place —
`MeshBuilder::addTriangleIndices` — and it was determined empirically rather
than assumed, with a probe program, because getting it backwards is not a crash.

Getting it wrong produces geometry that is *visible but lit from behind*: a roof
slope that stays grey in full sun, a lane marking that renders black on black
asphalt. Or geometry that is invisible from the only side anyone looks at it
from: a shop with no back wall, a car whose near-side panels are culled so you
can see the inside of its far side. All four of those happened here.

So `addQuadFacing(a, b, c, d, hint)` exists: where the intended facing is known
and the corner order comes from something else — a direction of travel, a side
of the street, five faces of a room in façade coordinates — state the facing and
let one function sort the order out. The room in a shopfront had its back wall
and ceiling culled precisely because a human derived five corner orders by hand
and got three of them right.

## Batching: the compromise nobody likes but everyone needs

Two things have to be true at once. Cheap to draw means as few draw calls as
possible, which argues for one enormous mesh per material. Cheap to cull means
the renderer can discard what is behind the camera, which argues for many small
meshes. A road batched as one 260 m mesh is one draw call that is always visible
and always fully rasterised.

`GeometryCollector` merges per material *within a 34 m grid cell* — a little
larger than a building plot, which is the natural granularity of a street. 42
buildings become a few hundred batches rather than tens of thousands of draws or
a handful of uncullable monsters.

Its one non-obvious property is documented in the header and is not a detail:
`builder()` returns a reference that stays valid. The generators hold several at
once — a wall, a frame and a pane of glass are interleaved while one window is
built — and an implementation that stored the builders directly in a
`std::vector` invalidated the first reference the moment the third material was
asked for. It did, and the result was a use-after-free that looked like a crash
somewhere else entirely.

## Neutral textures, tinted materials

The plaster, painted-metal, fabric and car-paint generators produce a *white*
surface: all the pattern, none of the colour. The colour arrives as the
material's base colour.

Seven façade colours cost one texture rather than seven. A fleet of ten
differently painted cars costs one texture set, not ten. Twenty painted-metal
variants — the signal housings, the lamp columns, the railings, the bins — cost
one. That is the difference between 40 MB of texture memory and 400 MB, and it
costs nothing visually, because the pattern really is the same on a green pole
and a grey one.

The exception proves the rule: the shop fascias each have a different name baked
into the board, so they genuinely cannot share, and they are the one place the
catalogue registers a material with textures of its own.

## Linear light, and the one place it is encoded

Every colour constant in the generators is quoted the way a paint chart quotes
it — as an sRGB byte triple — and converted to linear once, at the point it is
written. Textures are uploaded as sRGB and sampled to linear. Lighting,
tone mapping and post-processing are all linear. The encode back to sRGB happens
exactly once.

Where "exactly once" is depends on where the frame is going. `PbrEffect` encodes
its output to sRGB by default, which is right when it draws to the back buffer
and wrong when it draws to the HDR scene target, because the pipeline's tone
mapper encodes as well. With both, a 10:1 albedo ratio rendered as 1.6:1, and
the whole street looked washed out in a way that was very hard to attribute. The
renderer now tells the effect which case it is in, every frame.

## The sun is a composition decision

At 38° elevation, 17 m buildings shadow the entire 18.6 m canyon. The street is
uniformly in shade, there is no contrast anywhere, and it reads as sunless — and
the natural conclusion, when this happened, was that the shadow pass was broken.

It was not. Dumping the cascade atlas, logging the splits and differencing two
renders showed 21 % of pixels changing with shadows on and off. The shadows were
working perfectly. The scene simply had nothing lit to compare them against.

The default is 48° now, where the shadow reaches 15.3 m of the 18.6 m street and
the frame has both light and shade in it. This is the sort of thing that looks
like a rendering bug, is diagnosed like a rendering bug, and is a composition
decision.

## People: posed, not skinned, on purpose

A walking figure is generated at eight phases through one stride, and the
simulation picks the phase that matches how far the pedestrian has walked. At
1.35 m/s that is a new pose every 90 ms, which is smooth enough that nobody
counts them, and it costs two draw calls per person instead of the skinned
pipeline's per-bone state.

CNA *has* skeletal animation — `SkinnedEffect`, `AnimationClipEXT` — and with an
authored rigged character it would be the right tool. Generating a rigged mesh
procedurally, and a walk cycle to drive it, is a much larger piece of work whose
visible result at the distance a street is seen from is the same figure moving
the same way. The trade is deliberate and is stated in the header rather than
left for a reader to wonder about.

## Two placement loops that must agree, in one place

Lamps and street trees both go in the furniture zone beside the kerb. Two loops
each picking their own spacing will eventually plant a tree inside a lamp
column, and that is exactly the kind of mistake that survives every unit test
and then dominates a screenshot.

So the rhythm lives in one struct: lamps on a 24 m beat, trees on the 12 m
half-beat offset by six. The closest a tree ever gets to a column is 6 m — about
right for a real street, where the lighting engineer and the tree officer are
also obliged to talk to each other.

## Testing what has no picture

The unit tests are chosen for what a screenshot cannot see.

Both arms of the junction green at once, or a green man across a street whose
traffic is running, are *safety* properties: invisible in any still, and
catastrophic in motion. They are asserted at every sample of three whole cycles.

Two plots occupying the same ground is the defect that put a blank flank wall on
all four corners of the junction — visible in a screenshot, certainly, but only
if you happen to look at that corner, and trivially assertable.

A quad wound the wrong way is visible but lit from behind, which is exactly the
kind of wrong that survives review.

The framework is forty lines. What is under test is arithmetic and a state
machine; a failure needs to say which check failed, on which line, with the two
numbers that disagreed, and then stop. A framework dependency would say the same
thing and bring a build dependency with it.

## Changing CNA rather than working around it

The rule followed here: work around it in the application when the gap is this
project's problem, and fix it upstream when the gap is the framework's.

Eleven of the twelve findings in `docs/cna-findings.md` are worked around —
they are papercuts, documentation gaps, or things this project wants that the
framework has no obligation to provide.

The twelfth is different. `cna_tool_source_to_cnb` compiled every texture with
exactly one mip level, and the container had always been able to carry a chain.
The practical consequence is that compiling a scene's textures makes it look
*worse* than generating them at run time, which is not a trade anyone should
have to make, and no application-side workaround recovers it without throwing
away most of what the pipeline is for. That belongs in the codec, so it went
there — with the colour space as an argument rather than an assumption, because
averaging sRGB-encoded texels as if they were light darkens every level, and a
data map must be averaged exactly as it is stored.

## What "no fake complexity" meant in practice

The project deliberately does not have: a scene graph (a street does not move),
an entity-component system (there are four kinds of moving thing), a material
editor (there are 70 materials and they are in one enum), an abstraction over
CNA (there is one renderer and wrapping it would only add a place for the two to
disagree), or a plugin system.

It does have one enum with 70 entries, one dimension header, one mesh builder,
and generators that read like descriptions of the thing they build. Where there
is a class, it is because something owns state that outlives a function.
