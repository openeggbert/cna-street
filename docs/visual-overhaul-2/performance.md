# Performance, second pass

How it was measured, what the numbers are, and where the cost went. Every raw
number is in this file rather than a summary of them, because the machine was
not idle and a reader should be able to see how noisy the runs were.

## Method

`--frames 26` at viewpoint 1 (*Footway looking south to the junction*),
`--no-overlay`, the default `high` preset, Mesa llvmpipe under `xvfb-run` on a
16-core laptop: six warm-up frames discarded, twenty measured, the median
reported. The baseline binary is `27f92a8` built from a `git archive` into
`build-probe/baseline-src` with the same compiler, flags and compiled content
as the current one. The two binaries were run alternately in one session,
baseline first, two seconds apart, so that whatever the machine was doing hit
both.

It was doing a lot. Another agent was building and testing CNA on the same
machine throughout, alongside a browser and a Lisp image, and the one-minute
load average sat between 5 and 11 for every run. A software rasteriser
competes for exactly those cores, which is why single runs below differ by a
factor of two and why the table has eight pairs rather than one.

## Frame time, day, 1024 × 576

Median frame time in milliseconds over the twenty measured frames.

| Pair | Baseline `27f92a8` | Current | Note |
| --- | --- | --- | --- |
| 1 | 44.8 | 47.2 | |
| 2 | 35.6 | 73.6 | current run hit a burst: p95 124 ms |
| 3 | 34.6 | 74.5 | current run under load throughout: min 68 ms |
| 4 | 67.6 | 59.2 | both under load: p95 83 and 144 ms |
| 5 | 43.4 | 50.9 | load 10.5 / 9.3 |
| 6 | 38.2 | 41.6 | load 8.8 / 8.5 |
| 7 | 33.6 | 48.0 | load 7.9 / 6.7 |
| 8 | 42.9 | 41.5 | load 7.3 / 7.3 |
| **median of medians** | **40.6** | **49.5** | **+22 %** |
| **quietest run** | **33.6** | **41.5** | **+24 %** |

Stage breakdown from pair 6, the pair whose two runs had the smallest spread
between their own min and max:

| Stage | Baseline | Current |
| --- | --- | --- |
| cull | 0.22 | 0.29 |
| shadow | 14.76 | 16.25 |
| prepass | 2.50 | 2.86 |
| sky | 0.06 | 0.06 |
| opaque | 18.73 | 20.84 |
| transparent and post | 1.57 | 2.11 |

Every stage grows by roughly its share of the extra draws. The last row grows
the most in proportion because the glass now composites with `AlphaBlend` and
the district's windowed blocks add their panes to the transparent phase, but
it is half a millisecond.

## Night, and 1920 × 1080

One pair each, load 6.1 to 6.3 for all four runs.

| | Baseline | Current |
| --- | --- | --- |
| night, 1024 × 576 | 34.5 | 46.9 |
| day, 1920 × 1080 | 46.3 | 47.2 |

The 1080 pair says something the 576 pairs cannot. At three and a half times
the pixels the current build costs what it costs at 576, and the baseline only
twelve milliseconds more. Neither build is bound by pixels on this driver at
these sizes; both are bound by submission -- draws, state changes, batches --
and submission is what grew.

## What is fixed, and what grew

These do not depend on the load.

| | Baseline | Current | Change |
| --- | --- | --- | --- |
| Static batches | 1 187 | 1 586 | +34 % |
| Triangles in the scene | 1 042 354 | 1 072 898 | +3 % |
| Scene memory | 62 MiB | 65 MiB | |
| Draw calls per frame | 1 361 | 1 535 | +13 % |
| Shadow draw calls | 2 840 | 3 115 | +10 % |
| Triangles drawn per frame | 560 k | 603 k | +8 % |
| Scene build from compiled content | about 7 s | about 7 s | |
| Reflection probe bake | -- | 6.1 to 8.0 s over four runs | start-up, and again when the sun stops |

The batches are the district beyond: forty blocks with window recesses,
panes, sills, shopfronts, plinths and cornices in place of forty painted boxes,
and carriageway, footway and kerb faces continued to ±320 m. Each block's
materials batch per 34 m cell and the blocks are spread over 600 m of street,
so most of the four hundred new batches are theirs. The rest are the shop
decor tints, each of which is a material of its own, and the grime decals.

## The probes cost nothing per frame

Two interleaved pairs of the current binary, `--no-probes` against the
default:

| Pair | `--no-probes` | probes |
| --- | --- | --- |
| 1 | 45.7 | 50.4 |
| 2 | 50.2 | 48.6 |

Within the noise, in both directions, and the stage breakdown moves with the
load rather than with the flag. This is what was expected: `applyEnvironment`
rebinds three textures and two uniforms only when the probe changes between
consecutive draws, and a probe's cube is the same size as the sky's, so the
fragment work is identical. The whole cost is the bake at start-up: 29 probes,
six faces each at 64 px, each face a shadow-and-opaque render of the static
street into a `RenderTarget2D` and a `GetData` readback, then a prefilter and
an irradiance pass per probe through `EnvironmentProcessor`. The `medium`
preset halves the resolution and spacing (32 px, 32 m); `low` turns them off;
`--no-probes` skips the bake for a profiling run.

## Reading

About a fifth to a quarter more per frame at 1024 × 576, on a machine that was
never quiet, with the same figure at 1080 within noise. It buys reflections of
the actual street on every car and pane, a district that stays a street to the
horizon, forty shop interiors that read as rooms, lamps that sit in the car,
and a night sky. The first overhaul's own +15 % was measured the same way on a
quieter machine and is in `docs/visual-overhaul/performance.md`.

Where to get it back if it matters: the far blocks never move and are never
close, so they could batch per block instead of per cell; their panes could
become an opaque dark surface beyond 150 m and leave the transparent phase to
the near glass; and the far tree level of detail could become an instanced card
pair. None of it was done in this pass because none of it was visible.
