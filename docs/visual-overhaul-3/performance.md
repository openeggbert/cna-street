# What the third pass cost, and where

Every number here comes from `--frames 26`, which discards six warm-up frames,
collects the rest and prints mean, median, p95, min and max. The two builds --
`83dc8e1`, the commit before this pass, built into `build-probe/` from a
worktree with its own content, and the current tree -- were run interleaved in
one session, viewpoint 1, no overlay, on Mesa llvmpipe under Xvfb, which is
the same software rasteriser every earlier table in this project was measured
on. The machine was not idle: Blender, a content build and the screenshot
renders shared it during parts of the session, which is why there are three
pairs and not one number.

## Before and after

| | `83dc8e1` | this pass |
|---|---:|---:|
| median frame, 1024 × 576, three interleaved pairs | 52.0 / 63.8 / 55.4 ms | 47.2 / 52.1 / 58.6 ms |
| median frame, 1920 × 1080, one pair | 59.5 ms | 63.8 ms |
| draw calls per frame | 1 523 - 1 528 | 1 559 - 1 564 |
| shadow draw calls per frame | 3 115 - 3 125 | 3 291 - 3 303 |
| triangles drawn per frame | 588 k | 1 695 k |
| static batches at build | 1 634 | 1 634 |
| scene build | 10 s | 12 s |
| reflection probe bake | 7 s | 7–8 s |

Read the frame times as "within noise of each other, the new tree a little
heavier": the medians of the three 1024 × 576 pairs are 52.0 against 47.2,
63.8 against 52.1 and 55.4 against 58.6 ms, and the one 1080p pair is
59.5 against 63.8. Nothing in this pass changed the number of things drawn
by more than three per cent, and on this rasteriser the frame is bound by
submission rather than by pixels or triangles -- the earlier passes measured
1024 × 576 and 1920 × 1080 within noise of each other for the same reason.

What did change is the triangle count: roughly three times the triangles per
frame, almost all of them the hero trees at 158 000 apiece for the near level
of detail and 38 000 for the far one, with three or four of them in the
viewpoint-1 frustum. That is the visible cost of the one asset in the pass
that is geometry rather than texture, and it is spent in the fifty metres of
footway the showcase viewpoints stand on; every other tree in the district is
the generated one at a tenth of the weight. On the AMD integrated GPU this
machine actually has, where the interactive build was checked, triangles at
this count are not the bottleneck at all.

## Where the rest went

- **Textures.** The fourteen scans are 1024 px where the generated surfaces
  they replace were 256 or 512 (the road was already 1024), with full mip
  chains from the content compiler. Texture memory rose from 179 MiB to 194 MiB at
  start-up; the scans are all of the difference.
- **Scanned props.** Twenty models at 1k, instanced. The hydrant is 43 000
  triangles per copy after the aged variant is dropped, which is generous for
  a hydrant; everything else is under 20 000. They are instanced, and culled
  at a third to two thirds of the prop cull distance depending on size.
- **Scene build.** Two seconds longer: the thirty-one imported models load
  through `ContentManager`, the tree's two levels of detail among them at 15
  and 3.5 MB.
- **Nothing per frame for the light.** The bounce gain is one more cube per
  probe at bake time, convolved once; exposure and sun intensity are uniforms.

## What is still expensive

The same three things the second pass listed: the post chain on a software
rasteriser, the shadow pass drawing every instance one at a time (CNA-F6),
and skinned characters at three draws each. This pass added a fourth: the
hero tree at 158 000 triangles is more than a real-time tree wants to be, and
a leaf-card impostor level of detail past twenty metres would halve the
triangle count of every frame that looks down the street.
