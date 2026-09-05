# Before and after

Every pair below is the same viewpoint, the same seed and the same resolution
(1024 × 576), rendered by the same command with the same settings. BEFORE is
`c171cec`, the last commit before this pass; AFTER is the current tree.
Nothing in the pairs differs but the content: the camera, the clock, the seed,
the light and the anti-aliasing are identical -- the supersampled resolve is
*not* used for the pairs, so that what they show is the scene and not the
sampling.

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
| 17 | Pavement cafe | ![](before/17-pavement-cafe.png) | ![](after/17-pavement-cafe.png) |
| 18 | Covered car | ![](before/18-covered-car.png) | ![](after/18-covered-car.png) |

`pairs/` holds each of those as one image, before on the left and after on the
right, which is the form the differences are easiest to see in.

## The five that matter most

1. `pairs/09-car-three-metres.png` -- the white loft in the travel lane
   becomes an authored car, and so does every car behind it.
2. `pairs/16-corner-to-corner.png` -- three lofts turning through the
   junction become a VAZ estate, a Punto and a Logan; the woman in the
   foreground stands with her weight on one leg and her hands together.
3. `pairs/10-pedestrian-four-metres.png` -- the man at four metres walks
   with his knees bending the right way and an arm swinging, beside the
   woman he is walking with.
4. `pairs/11-shop-window.png` -- the cafe gets an oak floor, beams, panelling
   and a boarded counter; the pale plane that was the floor is gone.
5. `pairs/05-looking-up-at-the-facades.png` -- the upper floors get
   two-metre windows with curtains, blinds and dark rooms behind them, and
   keystones over their heads.

## Flagship

`flagship/` holds the eight flagship frames at 1920 × 1080: the footway
looking south, the car at three metres, the pedestrian at four, the shop
window, the street tree, the kerbside, the pavement cafe and the covered
car. Each is there twice: `vpN.png` at the same settings as the pairs, and
`vpN-ss2.png` through `--supersample 2`, which is the resolve the README's
screenshots are shot with from this pass on. `flagship-before/` is the same
eight frames from `c171cec`, at the pairs' settings.
