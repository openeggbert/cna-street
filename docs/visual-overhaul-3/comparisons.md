# Before and after

Every pair below is the same viewpoint, the same seed and the same resolution
(1024 × 576), rendered by the same command. BEFORE is `83dc8e1`, the last
commit before this pass; AFTER is the current tree. Nothing in the pairs
differs but the content and the light: the camera, the clock and the seed are
identical.

```
xvfb-run -a ./build/bin/cna-street --capture <dir> --width 1024 --height 576 --no-overlay
```

| # | Viewpoint | Before | After |
|---|-----------|--------|-------|
| 1 | Footway looking south to the junction | ![](before/01-footway-looking-south-to-the-junction.png) | ![](after/01-footway-looking-south-to-the-junction.png) |
| 2 | On the crossing | ![](before/02-on-the-crossing.png) | ![](after/02-on-the-crossing.png) |
| 3 | The corner block | ![](before/03-the-corner-block.png) | ![](after/03-the-corner-block.png) |
| 4 | Down the side street | ![](before/04-down-the-side-street.png) | ![](after/04-down-the-side-street.png) |
| 5 | Looking up at the façades | ![](before/05-looking-up-at-the-facades.png) | ![](after/05-looking-up-at-the-facades.png) |
| 6 | Above the junction | ![](before/06-above-the-junction.png) | ![](after/06-above-the-junction.png) |
| 7 | The long view south | ![](before/07-the-long-view-south.png) | ![](after/07-the-long-view-south.png) |
| 8 | Shopfronts, close | ![](before/08-shopfronts-close.png) | ![](after/08-shopfronts-close.png) |
| 9 | Car, three metres | ![](before/09-car-three-metres.png) | ![](after/09-car-three-metres.png) |
| 10 | Pedestrian, four metres | ![](before/10-pedestrian-four-metres.png) | ![](after/10-pedestrian-four-metres.png) |
| 11 | Shop window | ![](before/11-shop-window.png) | ![](after/11-shop-window.png) |
| 12 | Road surface | ![](before/12-road-surface.png) | ![](after/12-road-surface.png) |
| 13 | Street tree | ![](before/13-street-tree.png) | ![](after/13-street-tree.png) |
| 14 | Façade detail | ![](before/14-facade-detail.png) | ![](after/14-facade-detail.png) |
| 15 | Kerbside | ![](before/15-kerbside.png) | ![](after/15-kerbside.png) |
| 16 | Corner to corner | ![](before/16-corner-to-corner.png) | ![](after/16-corner-to-corner.png) |

`pairs/` holds each of those as one image, before on the left and after on the
right, which is the form the differences are easiest to see in.

## The photographs added by this pass

No BEFORE exists for these; they were added to stand on the stretch of footway
where the scanned content is densest.

| # | Viewpoint | After |
|---|-----------|-------|
| 17 | Pavement cafe | ![](after/17-pavement-cafe.png) |
| 18 | Covered car | ![](after/18-covered-car.png) |

## After dark

`--night` is unchanged by this pass except through the surfaces and props it
lights; it was rendered to check that the recalibrated daylight had not moved
it.

| Viewpoint | Night |
|-----------|-------|
| On the crossing | ![](night/vp2.png) |
| Kerbside | ![](night/vp15.png) |
| Pavement cafe | ![](night/vp17.png) |

## Flagship

`flagship/` holds seven frames at 1920 × 1080: the footway looking south, the
car at three metres, the shop window, the street tree, the kerbside, the
pavement cafe and the covered car.
