# Every external asset, and why it is here

Sixteen models, checked one at a time. `scripts/validate-assets.py` re-checks the
mechanical half of this on demand — fields present, licence on the allow-list,
attribution fields filled in where the licence needs them, digest and byte count
matching, nothing in `downloads/` that the manifest does not declare. It found an
undeclared 7.8 MB model in the fetched set on its first run, which is the whole
argument for having it.

The half a script cannot check is below: what each asset actually is, whether the
licence recorded for it is the licence *that model* carries rather than the one
its repository carries overall, and whether it earns its place in the scene.

## The source

All sixteen come from
[KhronosGroup/glTF-Sample-Assets](https://github.com/KhronosGroup/glTF-Sample-Assets),
fetched from `raw.githubusercontent.com`. That was partly a licence decision and
partly a network one: the egress proxy in this environment refuses Poly Haven,
ambientCG, Kenney, `github.com` HTML and `codeload`, and `raw.githubusercontent.com`
is what it allows. The constraint is recorded in `audit.md`; the *consequence* is
that the set is a materials showcase rather than a set of street furniture, which
is why the props behind the shop glass are a lantern, a boombox and a vase of
flowers rather than a mannequin and a shelf of shoes.

**The repository's licence is not each model's licence.** `Models.md` in that
repository states terms per model, and it carries models this project cannot use.
Three were considered and rejected:

| Rejected | Licence | Why not |
| --- | --- | --- |
| Duck | SCEA Shared Source License 1.0 | Not compatible with this repository's MIT terms |
| Sponza | CRYENGINE Limited Licence Agreement | Redistribution restricted |
| Virtual City | 3DRT, testing use only | Not licensed for use in a published work |

## The sixteen

| Local name | Title | Author | Licence | Fit to | Role |
| --- | --- | --- | --- | ---: | --- |
| `avocado` | Avocado | Microsoft | CC0-1.0 | 0.09 m | Greengrocer's display. |
| `boombox` | Boom Box | Ryan Martin / Microsoft | CC0-1.0 | 0.28 m | Electrical shop window. |
| `camera` | Antique Camera | Maximilian Kamps | CC0-1.0 | 0.32 m | Second-hand shop window. |
| `cesium-man` | Cesium Man | Cesium | CC-BY-4.0 | 1.8 m | The imported-skeleton round trip: compiled to .cnb by CNA's own glTF importer and loaded back at start-up, where its 19-bone skeleton, bind pose, i… |
| `chair-damask` | Chair Damask Purplegold | Eric Chadwick / Wayfair | CC-BY-4.0 | 0.92 m | Cafe seating, seen through the glass. |
| `chair-sheen` | Sheen Chair | Eric Chadwick | CC0-1.0 | 0.86 m | Cafe seating, second design. |
| `corset` | Corset | Microsoft | CC0-1.0 | 0.42 m | Clothing shop display. |
| `lantern` | Lantern | Ryan Martin / Microsoft | CC0-1.0 | 0.78 m | Ironmonger's window; also a hanging lamp inside a shop. |
| `plant` | Diffuse Transmission Plant | Rico Cilliers; Eric Chadwick | CC0-1.0 | 0.95 m | Planted interiors and the footway planters. |
| `pouf-silk` | Specular Silk Pouf | Eric Chadwick / Wayfair | CC-BY-4.0 | 0.42 m | Furniture shop window. |
| `refrigerator` | Commercial Refrigerator | Eric Chadwick / Wayfair | CC-BY-4.0 | 1.95 m | The convenience shop's chiller cabinet. |
| `sofa-leather` | Sheen Wood Leather Sofa | Eric Chadwick / Wayfair | CC0-1.0 | 0.78 m | Furniture shop window. |
| `sofa-velvet` | Glam Velvet Sofa | Eric Chadwick / Wayfair | CC-BY-4.0 | 0.84 m | Furniture shop window. |
| `sunglasses` | Sunglasses | Eric Chadwick / Wayfair | CC-BY-4.0 | 0.14 m | Optician's window display. |
| `vase-flowers` | Glass Vase with Flowers | Eric Chadwick | CC0-1.0 | 0.38 m | Florist's window and cafe tables. |
| `water-bottle` | Water Bottle | Ryan Martin / Microsoft | CC0-1.0 | 0.24 m | Convenience shop shelving. |

## What CNA refused, and why that is the right answer

Five of the sixteen are declined by the importer for extensions it does not
implement, and the content build warns and skips rather than failing:

| Model | Extension | CNA's position |
| --- | --- | --- |
| Sheen Chair, Sheen Wood Leather Sofa | `KHR_materials_sheen` | Not implemented; a file that *requires* it is refused rather than loaded with its sheen silently dropped |
| Specular Silk Pouf, Glam Velvet Sofa | `KHR_materials_specular` | Same |
| Materials Variants Shoe | glTF material variants | Same |

That is correct behaviour for a *required* extension. A renderer that loads a
file it cannot honour and draws it wrong is worse than one that says so — and
the scene has a fallback for every one of them, so five refusals cost five props
and no windows.

Two more are loaded and warned about rather than refused: the Sunglasses declare
`KHR_materials_transmission`, `KHR_materials_volume` and
`KHR_materials_iridescence`, and CNA approximates transmission as alpha blending
while saying in the warning exactly which four ways that is not physical. Both
behaviours — refusal for a required extension, an honest approximation with a
loud warning for an optional one — are worth the note.

## What was not used, and why

| Model | Reason |
| --- | --- |
| Diffuse Transmission Plant | 68 409 triangles, of which 54 000 are one part. A shrub with more geometry in it than the building it stands in; over the 22 000-triangle budget for a window display. |
| Cesium Man | Loads correctly — 19 bones, a bind pose and one clip — and does not draw through `SkinnedPbrEffect`. Kept and loaded at start-up so the round trip stays exercised, not placed. See `../cna-findings.md` GLTF-208. |
| Sheen Chair, Sheen Wood Leather Sofa, Specular Silk Pouf, Glam Velvet Sofa, Commercial Refrigerator, Materials Variants Shoe | Refused by the importer, above. Kept in the manifest because a rejected asset with a recorded reason is more useful than an absent one. |

## Transformations applied

Every asset gets the same two, and the manifest records them per asset:

1. Compiled to `.cnb` by `cna_tool_gltf_to_cnb` during the content build — CNA's
   own importer and content compiler, not a second interpretation.
2. Uniformly scaled at placement to the real-world size in `fitMetres`, because
   an authored asset's units are whatever its author felt like. The lantern
   arrives 29.98 m tall and the avocado 6 cm; both end up the size the shelf
   needs.

Nothing is edited, retextured, decimated or re-rigged. The originals are fetched
and verified byte for byte, and what the scene shows is what the author made.
