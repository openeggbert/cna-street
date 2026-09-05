# Before and after, second pass

Every pair below is the same viewpoint, the same seed, the same resolution
(1024 × 576), the same time of day and the same settings, rendered by the same
command. BEFORE is `27f92a8`, the commit this pass started from, rendered on
this machine with the compiled content and the imported props in place; AFTER
is the current tree.

```
xvfb-run -a -s "-screen 0 1400x800x24" \
    ./build/bin/cna-street --capture <dir> --width 1024 --height 576 --no-overlay
```

`pairs/` holds the seven pairs that show the most, side by side at half size,
for reading without switching between two files.

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

## What each pair shows

- **2, On the crossing.** The white saloon's tail lamps. Before: two red slabs
  the width of the boot, standing 14 cm off the bumper, because the lamp strip
  wrapped *out* behind the car and the brake lens was a box drawn over it.
  After: a cluster a hand tall wrapping the corner toward the car, a glossy
  lens with a little glow in it, and side glass that is tinted and reflects the
  buildings. The far end of the street is a street, with kerbs, trees and
  parked cars, rather than a wall of painted boxes.
- **9, Car, three metres.** The black car. Before: a dark shape. After: a
  horizon on the door, the facade opposite in the flank, the sky in the roof,
  and headlamps that are chromed reflectors behind a lens rather than two
  white squares. Nothing about the geometry of the paint changed; the
  environment it reflects did.
- **11, Shop window.** Before: a white box under two continuous light strips,
  with a shelf of confetti in it, seen through a milky filter. After: a room
  with a darker back, tube fittings in lengths, a poster, a counter, packaged
  stock, and glass that carries a faint reflection of the street at this
  near-normal angle -- which is the right amount at this angle.
- **8, Shopfronts, close.** The same rooms from the pavement, with the retractable
  awning that is now over part of the shop rather than none of it, and the
  reflection of the far facade in the panes.
- **1, Footway.** The shopfront glass on the left reflects the street across the
  road; the blue car's roof carries the sky and its side the facade; the far
  end closes on a street rather than on a set.
- **6, Above the junction.** The blocks lining the far street carry real
  window recesses and shopfronts; the far street has its kerb line, trees and
  parked cars. The skyline scatter beyond stays painted.
- **13, Street tree.** Before: a facade, because the viewpoint was aimed at
  where a tree was expected. After: a tree.
- **14, Façade detail.** The render no longer wears a dark dash every few
  centimetres. Whether a wall is streaked now depends on the plot, and where
  it is streaked depends on where the sills and downpipes are.

## The viewpoints added by this pass

| # | Viewpoint | What it is |
|---|-----------|------------|
| 15 | [Kerbside](after/15-kerbside.png) | A little below eye level along the parked cars with the shopfronts behind them, in a 35 mm field. The picture a person with a camera takes. |
| 16 | [Corner to corner](after/16-corner-to-corner.png) | Across the junction from the south-east footway, the signal and the crossing in front, the corner block behind. |

## After dark

`--night`, sun four degrees under the horizon, exposure 1.5. The sky is now a
twilight blue with the afterglow on the sun's side rather than a uniform dark
brown, the clouds take the sky's own colour, the flats' windows are lit in
ones and twos, and the shop windows and the lamps' pools light the pavement
they stand on.

| Viewpoint | Night |
|-----------|-------|
| Footway looking south | ![](night/01-footway-looking-south-to-the-junction.png) |
| On the crossing | ![](night/02-on-the-crossing.png) |
| The long view south | ![](night/07-the-long-view-south.png) |
| Shop window | ![](night/11-shop-window.png) |
| Kerbside | ![](night/15-kerbside.png) |

`pairs/02-on-the-crossing-night.png` puts the old night frame beside the new
one.

## Flagship

`flagship/` holds five frames at 1920 × 1080: the footway, the kerbside, the
car at three metres, the shop window, and the crossing at night. Everything
else here is at 1024 × 576 so a pair is a like-for-like comparison and the
repository does not carry a hundred megabytes of PNG.

```
xvfb-run -a -s "-screen 0 1920x1200x24" \
    ./build/bin/cna-street --viewpoint <n> --screenshot <file> --width 1920 --height 1080 --no-overlay
```
