# What the overhaul cost, and where

Every number here comes from `--frames N`, which discards six warm-up frames,
collects the rest, and prints mean, median, p95, min and max with the stage
breakdown averaged over the same window. Running the same command twice
reproduces them. The overlay's headline is an *exponential* average and its
breakdown is one frame's stage times: both are right for flying a camera around
and wrong for tuning, which is worth stating plainly because the first
performance note written during this work quoted the overlay at 214 ms on a
frame that took 778.

## The machine

There is one GPU available here and it is not a GPU: Mesa 25.2 llvmpipe, a
software rasteriser, on four cores. Every absolute number below is a property
of that. What survives the change of hardware is the *shape* — which stage
dominates, what a draw call costs relative to a triangle — and the shape on
llvmpipe is not the shape on a discrete GPU. Two differences matter when
reading this:

- A draw call here costs roughly what six hundred triangles do, because the
  per-draw work (state, uniforms, program) is CPU work and the rasterisation is
  also CPU work. On a GPU the ratio is far more lopsided in the draw call's
  favour, so the LOD work below would pay *more* there, not less.
- Full-screen passes are pure fill and scale exactly with pixels. At 1024 × 576
  post-processing is 40% of the frame; at 1920 × 1080 it is 67%. On a GPU it
  would be a few per cent.

Run-to-run variation on this machine is about ±60 ms at 1024 × 576, which is
larger than several of the individual wins below. Everything reported as a
comparison was measured by interleaving the two builds in the same session
rather than by running one and then the other.

## Before and after

`c4f80ce` is the last commit before the overhaul. Both builds render viewpoint 1
at 1024 × 576 with the `high` preset, 26 frames, no overlay.

| | baseline `c4f80ce` | current |
|---|---:|---:|
| median frame | 413–434 ms | 467–506 ms |
| p95 | 446–475 ms | 497–544 ms |
| draw calls | 1 159 | 1 356 |
| shadow draw calls | 2 269 | 2 819 |
| triangles drawn per frame | 531 045 | 559 349 |
| cull | 0.16 ms | 0.19 ms |
| shadow | 122.5 ms | 135.7 ms |
| depth/normal prepass | 26.8 ms | 31.2 ms |
| opaque | 109.2 ms | 120.9 ms |
| post | 178.4 ms | 203.4 ms |

Roughly **+15%** for: skinned animated pedestrians in place of rigid figures
baked at eight phases of a stride, twelve vehicle variants with lofted bodies,
interiors, separate wheels and working lamps, thirty-nine dressed shop
interiors, six tree variants with volumetric canopies, a distant city beyond
the modelled frontage, and a road map at four times the resolution.

## How it got there

The intermediate state was **+26%** — 449 ms against 568 ms — and finding the
cost took building the measurement rather than guessing at it. Drawn triangles
were up 11% while the frame was up 26%, which said immediately that this was
not a triangle problem. The report that found it prints the last frame's
*visible* set grouped by batch family and sorted by draw call:

```
cna-street:      351 draws    351 copies          0 tris  character  [skinned]
cna-street:       48 draws     48 copies       2880 tris  wheel-far-hatchback.0  [mover]
cna-street:       48 draws     48 copies       1536 tris  wheel-far-hatchback.1  [mover]
cna-street:       48 draws     48 copies       1440 tris  wheel-far-hatchback.2  [mover]
```

Of 1 997 draws, 351 were skinned characters and about 500 were wheels — a wheel
being three materials at four corners, so twelve draw calls per car for the
fact that wheels are round.

| change | draws before | draws after |
|---|---:|---:|
| Wheels baked into the distant body | ~500 | 0 |
| Character material set collapsed at distance (6 → 3) | 351 | 228 |
| Pedestrian cull distance of its own (210 m → 130 m) | 228 | 160 |
| Distant vehicle: trim folded into underbody, plate dropped | 46 | 0 |

Nothing above changes the silhouette of anything. The wheels stop rotating past
38 m, where a wheel is eight pixels across; a figure past 20 m loses the
material boundary between its shoes and its trousers, at which point it is fifty
pixels tall.

## Resolution

| | 1024 × 576 | 1920 × 1080 |
|---|---:|---:|
| median frame | 486 ms | 1 031 ms |
| shadow | 136 ms | 140 ms |
| opaque | 121 ms | 174 ms |
| post | 203 ms | 694 ms |

The shadow pass is resolution-independent — the cascades are 2048 px whatever
the window is — and the post chain is pure fill. Six full-screen passes at
2.07 megapixels on four CPU cores is 694 ms, and that is 67% of the flagship
frame. On any hardware with a rasteriser in it this line would be the smallest
in the table.

## Night

| | day | night |
|---|---:|---:|
| median frame | 486 ms | 503 ms |
| draw calls | 1 356 | 1 356 |

Night costs 3%, which is the light pools: forty extra blended quads and nothing
else. The lamps, the shop windows and the flats above them are emissive
materials on geometry that was already being drawn, so switching them on is
free. That is the whole argument for baking the street's light rather than
lighting it: `PbrEffect` carries one punctual light per draw and this street has
forty lamps, so a many-light path would have cost a different renderer.

## What is still expensive

- **Post processing**, at 40% of the frame at 1024 × 576 and 67% at
  1920 × 1080. Entirely a property of the software rasteriser.
- **The shadow pass**, at 2 819 draws for 1 356 visible objects, because CNA's
  cascade caster takes its world matrix from a uniform and has no instanced
  variant — see `docs/cna-findings.md` CNA-F6 — so an instance group is drawn
  one copy at a time into every cascade that reaches it.
- **Skinned characters**, at 160 draws for 53 people. Bone palettes cannot be
  instanced, so this is three draws per person and the only ways down are fewer
  people, a shorter cull, or a texture atlas that puts a whole figure on one
  material. The last is the right answer and is not done.
- **A per-material shadow-distance cap** was added and buys almost nothing on
  this scene: one plot's window frames are a single batch spanning a whole
  elevation, and a distance test against a batch takes its nearest corner, so an
  elevation running from 20 m to 60 m is kept whole. Per-batch culling can only
  ever be as fine as the batches, and these are coarse on purpose. The mechanism
  is right and the granularity is wrong; it is recorded here rather than removed
  because the next person to reach for it should know.
