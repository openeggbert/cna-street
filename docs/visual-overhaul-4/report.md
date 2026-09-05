# The fourth visual pass: hero assets and geometry

A fifth pass over `cna-street`, with the narrowest brief so far: not the
renderer, which after three passes has everything a street needs, but the
*things in it*. The third pass had put scanned surfaces and scanned street
furniture into the hero corridor and shown, by doing so, exactly what was
still wrong: the cars were lofts, the people were mannequins, the trees
beside the scanned one were blobs, the facades were planes with holes in
them, and the shops were boxes of packets. This pass replaced the objects a
viewer looks at first with authored, licensed ones, and spent its geometry
where the flagship cameras look.

Baselines, recorded before anything changed:

| Repository | Baseline | Note |
| --- | --- | --- |
| `cna-street` | `c31ae23` on `develop` | The commit this pass starts from |
| `cna` (`../cnanext`) | `next` `1704c32` | Read-only throughout |
| `sharp-runtime` (`../sharp-runtimenext`) | `next` `bfc826e` | Read-only |
| `easy-gl` | `develop` `deda7a4` | Read-only |
| `meta-gl` | `develop` `20c8b2d` | Read-only |

**No framework repository was modified.** Every change is in `cna-street`.
Two CNA behaviours were met on the way and are recorded in
`docs/cna-findings.md` as CNA-F15 and GLTF-209; both were worked around in
the asset pipeline and neither needed a change to CNA.

---

## 1. The five things wrong with the starting render

Rendered before changing anything, all eighteen viewpoints and the seven
flagship frames at 1920 x 1080, and looked at. In order of how quickly the
eye found them:

1. **The cars.** From three metres a loft is a loft: the body faceted along
   the roof, the greenhouse a tinted plane floating over the flank, the
   lamps flat rectangles, the wheels discs. The car at three metres was the
   first thing every flagship frame gave away.
2. **The people.** Tubes with a cap for hair and two dots for eyes, and the
   photographs had been framed to keep them out of the foreground.
3. **The trees beside the scanned one.** One scanned tree on the west
   footway, and beside it, across the road, and beyond it, the generated
   crowns: green noise-balls on sticks. The good tree made the bad ones
   worse.
4. **The facades.** Scanned brick and render on planes with sharp-edged
   holes cut in them. No arris on a sill, no depth to a shopfront, no
   bracket under a balcony, nothing on the wall that a person had fixed
   there.
5. **The shops.** Every unit a box lit by a ceiling of tubes with shelves
   of packets and a plinth. Through the glass at three metres that reads as
   a light box, whatever the trade.

## 2. What changed

### The cars (priority 1)

Eight authored car models, all CC-BY-4.0 from Sketchfab authors publishing
their own work, fetched from the Objaverse mirror with the licence, author
and page that Sketchfab embeds in the file itself: an Opel Astra GTC, a Fiat
Punto GT, a Renault Logan, a VAZ-2104 estate, a Honda Civic, a Mini Cooper
S, a generic 1980s saloon, and a Mercedes Sprinter van. No CC0 source has a
realistic car (Poly Haven has none), so the licence allow-list's CC-BY entry
did the work, and the attribution the licence asks for is in
`assets/ATTRIBUTION.md`.

`scripts/blender-vehicles.py` normalises each in Blender: the backdrop and
the baked shadow plane dropped, the hierarchy flattened, the model turned to
face +Z at its real length on y = 0, a body whose windows live only in a
texture's alpha split into an opaque body and a blended glass part, every
object joined into one mesh with one primitive per material, textures capped
at 1k (imported images carry one mip level, GLTF-206), a decimated far level
of detail, and the GLB's JSON patched so glass blends, everything else is
opaque, and no `KHR_materials_*` extension survives for CNA's importer to
refuse. `CityScene::buildHeroVehicles` deals them into the parked bays of
the main street within seventy metres of the junction, never the same model
in two neighbouring bays and never a car longer than the room its neighbours
leave it; the loft it replaces stays in the simulation and is not drawn.
They are static instance groups, so the reflection probes hold them and the
shop windows reflect them.

### The people (priority 2)

Eight people built from MakeHuman's CC0 base mesh and system asset pack --
skins, clothes, shoes, hair, eyes, eyebrows -- by `scripts/blender-people.py`
with MPFB in Blender, and written in this project's own character format
rather than glTF, because an imported skinned glTF part still draws nothing
through CNA's skinned path (GLTF-208) while the generator's own skinned meshes
draw fine. The script builds each person through MPFB's deserializer,
poses the arms down from MakeHuman's A-pose, bakes the targets and applies
the masks and the armature, folds MakeHuman's authored weights from its
137-bone rig onto the project's nineteen bones -- every finger onto the hand,
both twist bones onto the limb, the face onto the head -- and places the
nineteen joints at the rig's own joints. What comes out is a memory image of
what `SkinnedGpuMesh` uploads, plus the textures, which go through the same
compiler as the catalogue's surfaces and so arrive with a mip chain.

`CharacterLibrary` reads it; `CityScene` drops each person into the crowd
where a generated figure stood, on the same skeleton, driven by the same walk
and idle clips, with the same distance switch to a far copy and the same
rigid stand-in for the shadow pass. The generated figure stands in for any
variant without a person, so a tree without Blender still has a crowd.

### The vegetation (priority 3)

Two more scanned species from Poly Haven beside the small tree of the third
pass: a broad multi-stemmed island tree and a mature jacaranda, cut to a
street and a far level of detail by `scripts/blender-tree-lod.py`, which
learned to import a glTF and to strip the colour attribute a scan's LOD0
carries for its wind rig. The hero corridor is now the whole main street
within eighty metres of the junction, both footways: the west footway north
of the junction keeps the small tree with an island tree every third pit,
the east footway alternates the two bigger species, south of the junction
the two smaller ones alternate. The generated trees stand everywhere else.

### The architecture (priority 4)

`BevelBox`: a box on the wall with its four front edges chamfered, and
every sill, lintel, string course, cornice, pilaster, balcony nosing and
door threshold on the street is now made of it. The flat post-war and
contemporary elevations get a projecting chamfered surround round each
window instead of a sill, which is what those buildings have and what gives
a plane depth to measure. Balconies got a chamfered nosing and two brackets
with a strut. Every shopfront got full-height pilasters and corbels, and its
door set a third of a metre back into a proper opening with the wall
returning into the reveal, a chamfered stone threshold, and a pull handle on
two stubs. And the services a real wall carries: a meter cabinet by the
door, a louvred vent through the wall, a conduit run along the first-floor
line with saddles and a drop to a box, a scanned security camera over one
shop door in two, and scanned condenser units at the anchors the flat
elevations leave beside their upper windows.

### The hero storefront (priority 5)

One plot -- the shop the "Shop window" and "Pavement cafe" viewpoints look
into -- built as a composed bakery-cafe rather than a dressed box: a deeper
room with a front and a back and a darker store beyond a back door standing
ajar, a serving counter under a glass display case with the cakes and
croissants inside it and the till and a jug on top, bread shelving on the
back wall with baskets on a back-counter under it, a coffee station against
the end wall, three pendant lamps over the counter, a clock and prints on
the walls, a menu board, a low table in the window with the day's bread on
boards and in baskets, and two cafe tables between the window and the
counter. Outside: the awning always out, opening hours on the door glass, a
poster in the window, a lamp on the pilaster and a potted plant by the
door. Thirty-eight scanned props, every one anchored by the interior
generator where a person would have put it and stood up by the scene. The
fascia says KAFFEEHAUS.

### Micro detail (priority 6)

A sack truck left by the first delivery in the hero corridor, beside the
crates it brought.

## 3. Tried and rejected

- **Mercedes E-Class and Fiat 500 models from the same search.** Fetched,
  previewed, and dropped: the first for a provenance the author's page did
  not settle, the second for arriving as a CAD conversion. "It is CC-BY on
  the page" is not the whole test.
- **A per-material cull state for imported parts.** Considered as the fix
  for CNA-F15 and rejected in favour of reversing the winding in the derived
  files: the shadow pass, the prepass and the probe capture all set their own
  rasterizer state, and four places to keep in step is worse than one
  reversed index buffer.
- **Blended hair.** The skinned pass is opaque; masked hair with a low cutoff
  keeps the strips and costs nothing.
- **Higher-resolution car textures.** 2k looked better at three metres and
  shimmered at twenty, because an imported image has one mip level. 1k until
  GLTF-206 is fixed.

## 4. CNA capabilities used

Nothing new was called. What was used more fully: `cna_tool_gltf_to_cnb` on
sixteen more Blender-exported GLBs and twenty-four more Poly Haven documents;
`SkinnedPbrEffect` with `AlphaModeEXT::Mask` for hair and eyebrows;
`AnimationPlayer` on skinning data built from an authored skeleton rather
than a generated one; `ContentManager::Load<Texture2D>` for the people's
textures compiled with a mip chain.

## 5. Performance

See `performance.md`.

## 6. The frames that show it

- `pairs/09-car-three-metres.png` -- the Astra where the loft stood.
- `pairs/15-kerbside.png` -- the Logan and the Mini where two lofts stood.
- `pairs/10-pedestrian-four-metres.png` -- a person where a mannequin stood.
- `pairs/11-shop-window.png` -- the hero cafe where a box of packets stood.
- `pairs/01-footway-looking-south-to-the-junction.png` -- the whole change
  in one frame: the car, the trees, the people, the shopfront.
- `flagship/` -- at 1920 x 1080.

## 7. What still gives it away

1. **The people's faces, at two metres.** MakeHuman's skins are painted, not
   scanned, and at conversational distance a painted face is a painted face.
   At four metres, which is where a street is seen from, they hold.
2. **The imported textures' single mip level (GLTF-206).** Every scanned
   prop and every car is capped at 1k for it, and the cars' paint shows it
   at three metres.
3. **The moving traffic is still lofts.** Only the parked cars are authored;
   a car driving past the camera is the third pass's loft, because a moving
   car needs wheels that turn and the authored models do not separate them.
