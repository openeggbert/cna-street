# What the fourth pass cost, and where

Every number here comes from `--frames 26` at viewpoint 1, which discards six
warm-up frames, collects the rest and prints mean, median, p95, min and max
with the stage breakdown averaged over the same window, on Mesa llvmpipe
under Xvfb -- the same software rasteriser every earlier table in this
project was measured on. The baseline is `c31ae23`, measured at the start of
the session from the same build directory before anything changed; the
after numbers are three consecutive runs of the final tree on a machine that
was otherwise idle.

## Before and after

| | `c31ae23` | this pass |
|---|---:|---:|
| mean frame, 1024 x 576 | 46.5 ms | 59.9 / 63.6 / 61.5 ms |
| median frame, 1024 x 576 | 47.0 ms | 59.5 / 62.2 / 60.9 ms |
| mean frame, 1920 x 1080 | 63.8 ms (third pass's figure) | 75.7 ms |
| mean frame, `--night`, 1024 x 576 | -- | 55.5 ms |
| draw calls per frame | 1 566 | 1 700 |
| shadow draw calls per frame | 3 298 | 3 563 |
| triangles drawn per frame | 1.70 M | 5.15 M |
| static batches at build | 1 634 | 1 655 |
| scene triangles at build | 1.08 M | 1.51 M |
| scene mesh memory | 65 MiB | 88 MiB |
| scene build | 10.2 s | 16.9 s |
| reflection probe bake | 7 s | 8 s |
| catalogue textures | 215 at 194 MiB | the same |

Where the frame goes at 1024 x 576, before against after: opaque 22.4 against
30.4 ms, shadow 18.3 against 22.6, prepass 3.3 against 3.6, post 2.2 against
2.9, culling 0.3 against 0.4, sky free.

Read it as **about +30 %** at 1024 x 576 and +19 % at 1080p for eight authored
cars, eight authored people, nineteen scanned trees over three species, the
architectural detail on every facade, and a composed hero cafe. The draw
calls moved by nine per cent and the shadow draws by eight; the triangles
tripled, and on this rasteriser triangles are not free at this count -- the
opaque pass is where the extra eight milliseconds went, and the shadow pass
took four more for the same reason.

## Where the triangles went

The registered scene's heaviest families, by triangle, after the pass:

| family | copies | triangles registered | note |
|---|---:|---:|---|
| island tree, near | 9 | 1.8 M | 201 000 each; the far copy past 44 m is 56 000 |
| small tree, near | 7 | 1.3 M | 187 000 each; the far copy past 55 m is 45 000 |
| hydrant | 12 | 1.0 M | unchanged from the third pass |
| jacaranda, near | 3 | 0.8 M | 271 000 each; the far copy past 40 m is 90 000 |
| condenser units | 77 | 0.36 M | 4 700 each, culled at 67 m |
| Opel Astra GTC | 1 | 150 000 | the heaviest car; the Civic is 120 000 and the rest 27-42 000 |
| the people | 8 variants | 26-39 000 each | six draws each, as the generated figures were |

The trees are, as in the third pass, the item that would give the most back
on a real GPU with a leaf-card impostor level of detail past twenty metres,
and the item that costs least here relative to what it shows. The condenser
units were 160 copies at one bay in five before they were thinned to one in
twelve, which took a quarter of a million triangles off the scene for no
visible loss.

## Where the rest went

- **Textures.** The eight cars' images are capped at 1k by the Blender step
  (imported images carry one mip level, GLTF-206); the people's textures --
  a 2k skin and 1k garments -- go through the catalogue's compiler and carry
  a chain. The catalogue's own count and size are unchanged.
- **Scene build.** Seven seconds longer: sixteen car files, six tree files,
  twenty-three more Poly Haven props and eight people load through
  `ContentManager` and `CharacterLibrary`; the people's 68-byte vertices are
  built into their skinned buffers at start-up.
- **The probe bake.** One second longer, for the cars and the trees it now
  captures.

## What is still expensive

The same three things as before -- the post chain on a software rasteriser,
the shadow pass drawing every instance one at a time (CNA-F6), and skinned
characters at six draws each -- plus the tree triangles above. The people
cost no more draws than the mannequins did; what they cost is the skinned
vertex work of 30 000 triangles a figure where a mannequin was 5 000, which
the far copy at 35 % halves past the detail distance.
