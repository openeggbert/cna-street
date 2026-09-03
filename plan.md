# plan

Where the project is. Updated as it moves.

## Completed

**Phase 1 — audit.** Both dependencies read rather than guessed at: CNA's
module layout, its public XNA surface, the whole `CNAEXT` graphics layer, the
`PbrEffect` contract, the capability model, the CNB content pipeline and its
tools, and sharp-runtime's 44 components. Written up in `docs/cna-audit.md`,
with the baselines in `dependencies.lock`.

**Phase 2 — the build and the loop.** Modern CMake consuming CNA as a sibling
checkout, `CNA_CNAEXT` forced on, warnings scoped to this project's targets,
Linux first class, no IDE dependency and no absolute paths. A `Game` subclass,
a free camera, a settings document and a screenshot path.

**Phase 3–4 — the street.** `StreetMetrics.hpp`, the dimension book everything
reads. `CityLayout` generating a crossroads with eight frontages and 42 plots
from one seed. `RoadBuilder`: carriageway, junction box with corner fillets,
kerbs with a dropped profile at the crossings, footways, every marking, the
ironwork. `BuildingBuilder`: six architectural families, real window openings
with reveals and interiors, shopfronts, balconies, cornices, four roof styles,
rainwater goods.

**Phase 5–6 — surfaces and light.** `TextureFactory`: 30 authored procedural
surfaces in linear light, tileable by construction, with normal and ORM maps
derived from one height field. `SignFactory` and an SDF canvas with a stroke
font. `MaterialLibrary` with the neutral-texture/tinted-material strategy and
full mip chains. `SkySystem` on CNA's analytic model, feeding
`EnvironmentProcessor` for image-based lighting. `SceneRenderer`: cascaded
shadows, a depth/normal prepass, the HDR pipeline and its post chain, sorted
opaque and transparent draws, and a capability probe in front of every optional
subsystem.

**Phase 7–10 — the things in it.** `PropFactory` (lighting, seating, bins,
hydrants, cabinets, bike stands, planters, a bus shelter, trees and their
grates, signal heads and posts and masts, signs, street plates),
`VehicleFactory` (a lofted hull with a greenhouse and glass, five classes),
`PedestrianFactory` (posed figures at eight phases of a stride). `TrafficSignals`
as an eight-stage fixed-time controller, `TrafficSystem` as a car-following
model, `PedestrianSystem` as a walk graph that obeys the green man. All of it
placed by `CityScene`, with the shop fascias and house numbers lettered from
the anchors the façade generator leaves behind.

**Phase 11–13 — the frame.** Instancing for every repeated prop, frustum and
distance culling per batch and per instance with a shorter leash for shadows,
a debug overlay whose stage breakdown sums to the frame it measures.

**Phase 14 — the content pipeline.** `bake-assets --content` writing the
catalogue's surfaces as PNG through a device-less bake mode, a CMake `content`
target compiling them with `cna_tool_source_to_cnb`, and
`ContentManager::Load<Texture2D>` at start-up with a silent fallback to
generating. It needed a mip-chain generator contributed to CNA to be worth
having.

**Phase 15 — tests.** Eight suites, registered with CTest, over the logic that
has no picture. Two found live bugs on their first run.

**Phase 16 — documentation.** `README.md`, `docs/design-notes.md`,
`docs/cna-findings.md`, `assets/ATTRIBUTION.md`, and the screenshot set.

## Visual-quality work done

Each of these was found by looking at a screenshot, and each was fixed rather
than noted:

* Side-street frontages generated **inside** the corner blocks, which put a
  17 m blank flank wall on all four corners of the junction where four
  shopfronts should have been.
* Shop interiors drawn with the interior *atlas* — a 4×4 grid of whole rooms —
  stretched across the back wall, so every shopfront had a wall of thumbnail
  rooms behind the glass. And their back wall and ceiling culled, so a shopper
  looking in saw the sky through the back of the shop.
* Elevations that do not front a street built as blank walls, including the ends
  of terraces and the rear elevations visible from the junction.
* Vehicle hulls lofted inside out: every near-side panel culled, so a car looked
  like it was made of glass. Greenhouses whose glass was buried inside the
  pillar in the middle and floating in air at the ends. Wheels that were an
  uncapped tube with a bright disc inside — a hole with a hubcap floating in it.
  Bonnets two metres long.
* Asphalt aggregate at 13 cm per stone, so the carriageway looked like bubble
  wrap, and wheel tracks baked into a tiling texture, which put them *across*
  the carriageway repeating every three metres.
* Awnings built across a whole 20 m shopfront, roofing the footway.
* Chimney stacks five metres above the roof they sit on, from re-deriving the
  roof height instead of using the roof's own.
* Foliage cards with 24 cm leaves — a house plant hanging over the footway.
* Bollards eight to a crossing, which read as a car-park barrier.
* Signal lenses bright enough to bloom into a flare.
* Three viewpoints that were not places a person could stand: one inside a
  building, one in a running traffic lane a metre from a parked car.
* A sun at 38°, where 17 m buildings shadow the whole 18.6 m canyon and the
  street reads as sunless.

## Optimisation work done

* Per-material, per-cell batching: 787 static batches rather than one
  uncullable mesh per material or one draw per object.
* Instancing for every repeated prop; 408 instances in 54 groups.
* Frustum and distance culling per batch and per instance, with separate
  distances for drawing and for casting a shadow.
* Shared textures behind tinted materials: seven façade colours and a whole
  vehicle fleet's paint from one texture each.
* Full mip chains on every texture, without which a road at a grazing angle is
  a shimmer.
* Alpha masking instead of blending everywhere except glass, so nothing but
  glass needs sorting.
* The content pipeline, which takes the material stage from eight seconds to
  one.

## Asset work done

Every surface, sign face, letterform and mesh in the project is generated from
a seed by code in this repository. Nothing was downloaded, so there is nothing
whose licence has to be tracked; `assets/ATTRIBUTION.md` says so and says what
would have to happen if that ever changed.

## CNA issues found

`docs/cna-findings.md` has all twelve with reproductions, severities and
workarounds. The one that was fixed rather than worked around is CNA-F12: the
content pipeline could not produce a mip chain, which made compiling a scene's
textures look worse than generating them. `docs/patches/` carries the commit.

## Next

* **Reflections.** Screen-space reflections are wired to a setting and off. A
  shop window that reflects the car parked in front of it is a visible step up
  at eye level.
* **Skinned pedestrians.** The posed figures are a deliberate trade, but an
  authored rigged character through `SkinnedEffect` and `AnimationClipEXT`
  would exercise a part of CNA this does not.
* **Turning traffic.** Four straight lanes is the right level of detail for
  environmental traffic, but a right turn across the junction would make the
  signal phases mean more.
* **Audio.** CNA's audio module works; the demo has nothing to play through it.
* **A measured `low` preset.** Its decisions are reasoned rather than measured,
  because this environment has only a software rasteriser.
* **Image-based screenshot regression.** The viewpoints are stable and
  `--capture` is deterministic, so the comparison is the missing half.

## Blockers

None.
