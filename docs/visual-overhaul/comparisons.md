# Before and after

Every pair below is the same viewpoint, the same seed and the same resolution
(1024 × 576), rendered by the same command. The BEFORE column is
`c4f80ce`, the last commit before this overhaul began; the AFTER column is the
current tree.

```
xvfb-run -a ./build/bin/cna-street --capture <dir> --width 1024 --height 576 --no-overlay
```

The six viewpoints after the eighth did not exist before the overhaul. They
were added because the original eight were all chosen from a comfortable
distance, and a street that survives being looked at from ten metres does not
necessarily survive being looked at from three. They have no BEFORE column
because there is nothing to compare them with; what they show is in
`flagship/` at 1920 × 1080.

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

## The viewpoints added by the overhaul

Each one is aimed squarely at something that used to be a weakness, from the
distance a person would actually see it from, and none of them is a forgiving
angle.

| # | Viewpoint | What it is there to expose |
|---|-----------|----------------------------|
| 9 | [Car, three metres](after/09-car-three-metres.png) | Body surface, glazing, wheels, lamps, panel gaps |
| 10 | [Pedestrian, four metres](after/10-pedestrian-four-metres.png) | Human silhouette, clothing, gait |
| 11 | [Shop window](after/11-shop-window.png) | Glass, what is behind it, and the interior's own light |
| 12 | [Road surface](after/12-road-surface.png) | Aggregate, markings, kerb line, at the angle that exposes tiling worst |
| 13 | [Street tree](after/13-street-tree.png) | Trunk, branch structure, leaf silhouette, from underneath |
| 14 | [Façade detail](after/14-facade-detail.png) | Reveal depth, sill, material scale, one bay filling the frame |

## After dark

`--night` puts the sun four degrees below the horizon and switches the street's
own lights on. There is no BEFORE column because there was no night.

| Viewpoint | Night |
|-----------|-------|
| Footway looking south | ![](night/01-footway-looking-south-to-the-junction.png) |
| On the crossing | ![](night/02-on-the-crossing.png) |
| The long view south | ![](night/07-the-long-view-south.png) |
| Shop window | ![](night/11-shop-window.png) |

## Flagship

`flagship/` holds five frames at 1920 × 1080 — the two street views, the car,
the shop window and the junction at night. Everything else in this directory is
at 1024 × 576 so that a before/after pair is a like-for-like comparison and so
that the repository does not carry ninety megabytes of PNG. The full set at
1920 × 1080 is one command away:

```
xvfb-run -a -s "-screen 0 1920x1200x24" \
    ./build/bin/cna-street --capture <dir> --width 1920 --height 1080 --no-overlay
```
