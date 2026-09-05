# What the fifth pass cost, and where

Every number here comes from `--frames 26` at viewpoint 1, which discards six
warm-up frames, collects the rest and prints mean, median, p95, min and max
with the stage breakdown averaged over the same window. Two rasterisers this
time: Mesa llvmpipe under Xvfb, which every earlier table in this project was
measured on, and -- because this pass was the first one somebody ran on the
window rather than on the capture -- the machine's own integrated GPU, an AMD
Radeon 780M, on the desktop. The baseline is `c171cec`, built into the same
build directory from a stash of this pass's changes and run against the same
compiled content, so the two binaries differ only in what this pass changed.

## Before and after, llvmpipe

Three runs each, interleaved in one session on a machine that was running
other builds throughout (load average 8-14), which is why every number here
is higher than the fourth pass's table and why the comparison is between
these two columns and not against that table.

| | `c171cec` | this pass |
|---|---:|---:|
| mean frame, 1024 x 576 | 73.2 / 86.2 / 77.8 ms | 75.6 / 73.9 / 79.1 ms |
| median frame, 1024 x 576 | 72.3 / 86.1 / 79.1 ms | 75.6 / 71.3 / 77.3 ms |
| draw calls per frame | 1 775-1 777 | 1 753-1 783 |
| shadow draw calls per frame | 3 814-3 820 | 3 522-3 528 |
| triangles drawn per frame | 5.15 M | 5.61-5.64 M |
| static batches at build | 1 655 | 1 721 |
| scene triangles at build | 1.51 M | 1.56 M |
| scene mesh memory | 88 MiB | 94 MiB |
| scene build | 29 s | 27 s |
| reflection probe bake | 11 s | 10 s |

Where the frame goes at 1024 x 576, before against after: opaque
38.2-43.1 against 39.5-42.2 ms, shadow 3 814-3 820MS against
25.3-27.2, prepass 4.3-5.5 against 4.6-5.0, post 3.0-3.5
against 3.4-3.9, culling 0.5 either way, sky free.

Read it as **within noise**: the three-run means average 79.0 ms before and
76.2 ms after on a loaded machine, with the shadow draws down eight per
cent and the triangles up nine. The authored traffic costs what the lofts
cost, once its wheels stop being separate draws past thirty-two metres and
the shop props stop casting shadows nobody can see; the dressed windows,
the shutters and the quoins are a few thousand triangles in batches that
were being drawn anyway.

## What the first version of this pass cost, and what was taken back

The first full measurement of this pass came out at 92 ms with 1 809 draws
and 4 070 shadow draws: every authored car on the move is one draw per
material plus two or three per wheel -- the small hatchback has nineteen
materials -- and every scanned prop stood in a shop was casting shadows
inside a room the sun never reaches. Two changes took most of it back
before the numbers above were taken: the props inside shops cast no
shadow (five hundred shadow draws), and a moving car switches to its
welded far copy at thirty-two metres rather than forty-five, where the
wheels' eight to twelve draws stop being paid. The potted plant, a 176 000
triangle scan, was cut from every shop window to the florist's and the
cafe door. What remains over the baseline is the authored traffic itself,
and it is what the frames show.

## Where the triangles went

The registered scene's heaviest families after the pass, by triangle:

| family | copies | triangles registered | note |
|---|---:|---:|---|
| island tree, near | 9 | 1.8 M | unchanged |
| potted plant | 9 | 1.6 M | 176 000 each; florists and the cafe door, culled at 24-28 m |
| small tree, near | 7 | 1.3 M | unchanged |
| hydrant | 12 | 1.0 M | unchanged |
| jacaranda, near | 3 | 0.8 M | unchanged |
| condenser units | 23 | 0.44 M | unchanged |
| wicker basket | 15 | 0.33 M | 22 000 each, in the cafe and the bakeries |
| Opel Astra GTC | 1 parked, plus the traffic | 150 000 per near copy | 33 draws a copy; the far copy is 18 000 |

## The GPU

The user ran this pass's build on the window for the first time and saw
nine frames a second at 1600 × 900 with everything on, thirty with shadows,
SSAO, bloom, fog and clouds off (F2 to F6). Measured with `--frames 40` at
the same size on the same desktop -- an AMD Radeon 780M through Mesa
25.0.7, with the user's own window still open on the same GPU and the
machine at a load average of thirteen from other builds, so these are
upper bounds on the frame and not a benchmark:

| | frame | shadow | prepass | opaque | post | draws | shadow draws |
|---|---:|---:|---:|---:|---:|---:|---:|
| everything on | 88.9 ms (11.3 fps) | 30.9 | 5.3 | 47.7 | 4.4 | 1 785 | 3 504 |
| `--no-shadows` | 48.1 ms (20.8 fps) | 0 | 4.5 | 39.4 | 3.6 | 1 778 | 0 |
| shadows, SSAO, bloom, fog and clouds off | 45.8 ms (21.8 fps) | 0 | 0 | 41.4 | 3.8 | 1 777 | 0 |

Two things are legible in that even under the noise. The post chain,
SSAO and the prepass together are a few milliseconds on this GPU: turning
them off buys nothing worth having. The shadow pass is forty per cent of
the frame, and the opaque pass -- 1 780 draws for 5.6 million triangles
-- takes forty milliseconds with the pixels a rounding error, which is
the signature of a frame bound by draw submission and not by shading:
about twenty microseconds a draw through CNA's OpenGL 3.3 path, nine a
draw in the shadow pass, where each instance is its own draw (CNA-F6).
The stage times are wall-clock around the submissions, so on a GPU they
measure the CPU handing work over, and this machine's CPU was busy.

What this project can do about it on its own side: fewer draws. The
skinned people are three hundred draws a frame (six per figure) and the
largest single family; an atlas per figure would make them fifty. A
moving authored car is up to thirty-three draws because its author gave
it up to nineteen materials; merging the small ones in the Blender step
would halve that. Both are on the list. What it cannot do: the per-draw
cost itself, and the shadow pass drawing every instance one at a time,
are the framework's (CNA-F6), and per the brief nothing in CNA was
changed for this pass -- this table is the report.
