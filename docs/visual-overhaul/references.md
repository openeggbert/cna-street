# What this street was aimed at

A rendering is judged against something, and if that something is unstated it is
whatever the person who wrote the shader happened to picture. This file states
it.

## The place

A continental European city street of the kind you find in a *Gründerzeit*
perimeter block — Germany, Austria, Czechia, northern Italy — where the ground
floor trades and the four storeys above it are flats. The signage is German
(`BÄCKEREI`, `APOTHEKE`, `SCHUHE`, `GALERIE`) and the road markings and signs
are the German set, because a street has to be *somewhere*: a generic one is a
set of averages and averages do not look like anything.

Concretely, the reference is a street about 22 m building line to building line,
with a 11 m carriageway between 3.8 m footways, four-to-five storey blocks in
lime render and brick, plane trees on a 12 m pitch in the footway, and a
signalised crossroads at one end.

## The material references, and what each one is for

These are descriptions rather than photographs — `TextureFactory.cpp` is the
place where each is written down as numbers — but they are the things the
numbers were chosen to match.

| Surface | Reference | The number that matters |
| --- | --- | --- |
| Asphalt | AC 11 D surface course, four or five years old | Bitumen film is 3–5% reflectance; exposed granite chip is 8–15%. Two or three to one, not ten. |
| Aggregate | 8–16 mm grading, rolled flush | A 3 cm chip needs 0.5 cm per texel to be drawn at all, which is what set the road's map size and tile |
| Footway | 50 × 50 cm concrete *Gehwegplatte* on sand | 2 mm of lippage between slabs — grazing light finds it from thirty metres |
| Kerb | Granite, 14 cm upstand | |
| Gutter | Two courses of 10 cm setts | |
| Render | Lime render, repainted, weathered | Rain streaks run *down from* sills and *up from* the plinth, not evenly |
| Brick | 240 × 71 mm with a 10 mm bed joint | The joint is 4% of a brick's height and reads at 30 m; get it wrong and the wall reads as tile |
| Roof | Clay pantile, zinc standing seam, mineral felt | |
| Car paint | Two-coat metallic under clearcoat | Clearcoat is about 0.085 roughness. 0.13 is a matte finish; 0.05 puts a white disc of sky on every roof. |
| Car glass | Tinted, transmitting about a quarter at street angles | The rest of what reaches the eye is reflection, which is why alpha 0.52 gave a fleet of greenhouses |
| Tyre | 205/55 R16 | Rim is 0.635 of the tyre's radius |
| Skin, clothing | | |

## The proportions

The figures are built from adult anthropometry as fractions of stature, because
that is the one shape every viewer is an expert in:

| Landmark | Fraction of stature |
| --- | --- |
| Ankle | 0.040 |
| Knee | 0.285 |
| Hip | 0.530 |
| Wrist | 0.468 |
| Elbow | 0.630 |
| Shoulder | 0.812 |
| Neck | 0.828 |
| Head centre | 0.905 |
| Crown | 0.998 |
| Biacromial half-breadth | 0.1125 |

A head is about 0.155 m wide on a 1.75 m figure and shoulders about 0.39 m. When
those two ratios are wrong the figure reads as a mannequin no matter what else
is right — which is exactly what happened when a modelling error made the
shoulders *look* pinched and the head therefore look enormous.

## What the viewpoints are aimed at

Six of the fourteen viewpoints exist because the original eight were all chosen
from a comfortable distance. Each new one is aimed at something that used to be
a weakness, from the distance a person would actually see it from:

| Viewpoint | Distance | The question it asks |
| --- | --- | --- |
| Car, three metres | 3 m | Does this read as a pressed steel body or as a box with a texture? |
| Pedestrian, four metres | 4 m | Does this read as a person? |
| Shop window | 1.5 m | Is there anything behind the glass, and is it lit like a shop? |
| Road surface | 0.4 m, grazing | Is this asphalt, and does the tiling show? |
| Street tree | under it | Is this a tree or a coat rack with leaves on? |
| Façade detail | one bay | Is the reveal deep, the sill real, the brick the right size? |

## The test

> Would a stranger, shown a still from this, say *"a tech demo with programmer
> assets"* or *"a surprisingly convincing rendered city street"*?

Not *photorealistic*. Convincing — which is a lower bar and a more useful one,
and which fails on exactly the things this file is a list of.
