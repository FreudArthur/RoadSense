# RoadSense Annotation Guide

Version 0.2, 2026-10-02. These rules make every image annotated the same way. If a rule changes, log the date and the reason in the changelog at the bottom.

## 1. What is annotated

One mask class: **`road`**. A second CVAT label, **`void`**, is used only to cut holes in the road polygon (see section 3).

> **Road = any flat, continuous ground surface on which a motorbike or a car can normally drive, whatever the material** (asphalt, pavers, dirt, sand, gravel).

A dirt track is a road. Pavers covered with sand are a road.

## 2. Included in the mask

- The full width of the drivable way, up to its boundary (section 4).
- Puddles on the road.
- Tyre tracks, leaves, small stones and litter lying flat on the road.
- Flat dirt areas in front of houses and shops **if they are continuous with the road** (no step, curb or obstacle).

## 3. Excluded from the mask

- **The camera holder's own vehicle and body**: handlebar, speedometer, mirror, arms, feet, the driver in front (passenger view).
  When such an object sits **inside** the road (typically the mirror), draw the road polygon normally, then draw a `void` polygon around the object. `void` is drawn last so it is on top, and it becomes background in the final mask.
- Anything standing or parked on the road: motorbikes, cars, pedestrians, animals. Only the visible road surface is labelled.
- Raised sidewalks and anything beyond a curb or gutter.
- Piles of sand, gravel or rubbish that rise above the ground.
- **Stones and bricks, the rule of thumb:** would a motorbike ride over it without slowing down? Yes (pebbles, a flat broken brick, small stones) → include in `road`. No, it would steer around it (a whole brick standing up, a large rock, a pile of bricks or rocks, a tyre) → exclude.
- **Grass, same logic:** sparse short tufts growing on the driven surface → include. A continuous grass patch, an island between two branches, or the grassy roadside → exclude.
- **Tiny far-away objects** (smaller than a fingernail on screen): do not cut them out, just let the polygon pass under them.
- Grass, bushes, stalls, walls, buildings, sky.

## 4. Where the road ends

Stop the mask at the first of:

1. a curb, gutter or step;
2. a wall, fence or building;
3. vegetation (tall grass, bushes);
4. a stall, a pile of material, a fixed obstacle.

**Fuzzy boundary** (road dirt blends into roadside dirt): follow the edge of the compacted area (tyre tracks, smoother ground). When in doubt, stay slightly inside and set `ambiguous_boundary`.

**Far away in the image**: annotate as long as the road is clearly visible. Do not guess beyond.

## 4b. Several surfaces in one image

**The mask does not change**: the class is `road`, the material does not matter. The only question is:

> **Is the sand next to the pavers part of the way vehicles use?**

| Situation | Decision |
| --- | --- |
| Sand at the same level as the pavers, no curb, tyre tracks or bikes on it | Inside the `road` mask |
| Sand behind a curb, gutter or step | Outside the mask |
| Sand strip at the same level but unused (no tracks, objects on it, grass growing) | Outside the mask, set `ambiguous_boundary` |
| Sand covering the pavers in places | Inside the mask, same road |

What changes is the **`surface` tag**. Principle: **label what you see, not what you know.** The model only sees appearance. Look at the lower half of the image and answer in order:

1. Two clearly distinct zones side by side (e.g. pavers in the centre, sand strip on the side), each roughly a third of the road or more → `mixed`. If the other zone is small, ignore it and go to question 2.
2. Pavers clearly visible over most of the road → `pavers`.
3. Pavers covered with sand, joints or some pavers still visible → `sand_covered_pavers`.
4. So much sand that no paver is visible → `unpaved`, even if you know pavers are underneath. Write `pavers_underneath` in `note`.
5. Smooth grey bitumen → `asphalt`.
6. Impossible to decide (night, blur, unrecognisable surface) → `uncertain`, with the reason in `note`.

Three-second rule: decide at first glance. Consistency matters more than perfection. If more than 1 image in 20 ends up `uncertain`, a rule needs revising.

## 5. Per-image tags (`frames/manifest.csv`)

| Column | Values | Rule |
| --- | --- | --- |
| `surface` | `asphalt`, `pavers`, `sand_covered_pavers`, `unpaved`, `mixed`, `uncertain` | Decision tree in 4b |
| `lighting` | `day`, `dusk_night` | Derived from video time |
| `weather` | `dry`, `rain` | `rain` = filmed while raining (wet, reflective road) |
| `note` | free text, `;`-separated | e.g. `passenger_view`, `on_foot`, `occluded`, `low_angle`, `gravel`, `pavers_underneath`, `ambiguous_boundary`, `motion_blur`, `pedestrian_walkway`, `shadows`, `sand_dusted` (sand on asphalt), `tilted`, `gravel_shoulder`, `ground_only` (camera pointed at the ground, no horizon) |
| `surface_user` | earlier labels by the annotator | Kept to measure agreement between two labelers |
| `tag_source` | `viewed`, `propagated` | `propagated` = copied from the nearest viewed frame of the same video (1 s apart) |
| `split` | `test`, `train_seed`, `train_preannotate`, empty | See section 7 |

## 6. Examples from the dataset

| Image | Case | Decision |
| --- | --- | --- |
| `20261001_143614_f001950` | Pavers almost entirely covered with sand | `road` everywhere between curbs, `sand_covered_pavers`. Exclude bikes and car |
| `20261001_143236_f000980` | Dirt track with leaves and a gravel patch | Dirt and gravel included, leaves included. Stop at the grass on the right. Exclude pedestrians |
| `20260930_073737_f000957` | Dirt, puddle, low wall on the left, shops | Puddle included. Stop at the wall. Flat area in front of shops included if continuous |
| `20260930_190053_f000840` | Dirt at dusk, dark and slightly blurry | Annotate normally, `dusk_night`, `motion_blur` |
| `20261001_074350_f000522` | Passenger view, driver's helmet and yellow shirt in frame | Exclude driver, helmet, handlebar and mirror |
| `20261001_143614_f003030` | Paved walkway between campus buildings | `road`, surface `pavers` (decided 2026-10-02) |

## 7. Annotation protocol and splits

- Frames live in two folders, each with its own `manifest.csv` and `orientation.csv`: `frames/` (videos of 2026-09-30 and 2026-10-01) and `asphalt/frames/` (asphalt videos of 2026-10-03).
- **`test`** (50 + 14 asphalt images): annotated **from scratch**, never pre-annotated. Taken from videos or time blocks that never appear in train, so near-duplicate frames cannot inflate scores.
- **`train_seed`** (51 + 8 asphalt images): annotated from scratch. Used to train a small model that produces draft masks.
- **`train_preannotate`** (96 + 16 asphalt images): draft masks from that small model, then corrected by hand. Draft masks must **never** come from the off-the-shelf models being evaluated, otherwise the ground truth would be biased towards their predictions.
- SAM click-based tools are allowed everywhere: they have no notion of "road", the annotator decides every region.

## 8. Output format

- PNG masks, same file name as the image: `0` = background, `1` = road.
- Never modify the original images.

## 9. Decisions on edge cases

- Paved pedestrian walkways between buildings (`pedestrian_walkway`): **included as `road`**, surface `pavers`. A motorbike can physically ride there, and the surface is identical to paved streets. The `pedestrian_walkway` note is kept so these images can be analysed separately.
- `sand_covered_pavers` is a **surface tag, not a mask class**. Masks stay binary (`road` / background). The tag lets us test whether models still recognise a road when its texture is hidden under sand.

## 10. Changelog

- 2026-10-01: initial version (French).
- 2026-10-01: rule 4b (several surfaces), `mixed`, decision tree, `uncertain`, `note`.
- 2026-10-02: translated to English, surface values renamed (`unpaved`, `pavers`, `sand_covered_pavers`, `asphalt`), splits and protocol added.
- 2026-10-02: pedestrian walkways included as `road`; `sand_covered_pavers` confirmed as surface tag.
- 2026-10-03: grass rule (sparse tufts included, patches excluded).
- 2026-10-03: rule of thumb for stones and bricks, tiny far-away objects not cut out.
- 2026-10-03: `void` label to cut ego-vehicle parts (mirror) out of the road.
- 2026-10-03: asphalt videos added (`asphalt/frames/`), `weather` column, new note values. Rain asphalt is test-only (unseen condition at training time).
