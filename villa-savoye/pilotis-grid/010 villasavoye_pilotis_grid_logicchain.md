# Villa Savoye: Parametric Pilotis Grid

## Logic Chain Report

| Item | Detail |
| --- | --- |
| **Definition file** | `VillaSavoye_Pilotis_Grid.gh` |
| **Software** | Rhinoceros 8 (Grasshopper 1.0) |
| **Units** | centimeters |
| **Footprint** | 1920.41 × 2150.39 cm (referenced rectangle curve) |
| **Status** | Phase 1 done: grid generation and culling verified |
| **Next** | Column bodies (separate definition), then façade-offset and tributary-area metrics. Possible later extension: a choice between edge-anchored and centred grids |

**Scope in one sentence.** This is a generator for rectangular, axis-aligned pilotis grids, anchored at the west and south edges of the footprint. It doesn't handle rotated or irregular footprints yet, and it says nothing about structural adequacy.

**To reproduce.** Open `VillaSavoye_Pilotis_Grid.gh`, reference the 1920.41 × 2150.39 cm footprint rectangle in the `Crv` parameter, set both Bay Spacing sliders (X and Y) to 300, and leave the two constants at 2. You should get 72 candidate points and 56 surviving columns. Appendix B lists the checkpoint values along the way.

---

## 0. What this definition does

You give it one closed rectangle, the Villa Savoye footprint, and it hands back a column grid. Two sliders, Bay Spacing X and Bay Spacing Y, set the bay size on each axis. Any column that lands outside the footprint gets thrown away. In total the definition proposes 72 candidate points and keeps 56.

That's the whole trick. Getting there took thirteen components and seven bugs, and the rest of this report walks through both.

```text
footprint curve
   └── Deconstruct Rectangle ──► plane + X/Y intervals
         ├── Deconstruct Domain ──► start / end per axis
         │       └── Subtraction ──► full width / height
         │             └── Integer Division ──► whole bays that fit
         │                   └── Addition (+2) ──► point count per axis
         │                         └── Series ──► list of coordinates per axis
         │                               └── Cross Reference ──► 72 (X, Y) pairs
         │                                     └── Construct Point ──► points (at world origin)
         │                                           └── Move (by plane origin) ──► points on the footprint
         │                                                 └── Point In Curve ──► True/False per point
         │                                                       └── Cull Pattern ──► 56 surviving columns
         └── Deconstruct Plane ──► plane origin (the Move translation)
```

Read it top to bottom. The left spine does the arithmetic; the single branch on the right (`Deconstruct Plane`) only exists to carry the building's center point down to the `Move` step.

---

## 1. The footprint becomes a reference frame

**Components:** `Curve` (Params → Geometry), `Deconstruct Rectangle` (Curve → Primitive)

I don't copy the rectangle. I take it apart. `Deconstruct Rectangle` splits it into a plane and two intervals, and from that point on those two things do all the work.

The plane says *where the building is*. Its origin is the rectangle's center, a "you are here" pin. The intervals say *how far it reaches*, one from-to range per axis. Every calculation downstream happens on the intervals, and every placement happens relative to the plane. Most of the definition is just translating between the two.

Panel readout:

```text
X:  −960.204105 To 960.204105      (full width  = 1920.408 cm)
Y:  −1075.194692 To 1075.194692    (full height = 2150.389 cm)
```

Notice these are ranges, not widths. They run negative because the rectangle was drawn from its center in Rhino, so every coordinate in this report is measured from the middle of the building, the way a map measures from its own origin.

---

## 2. Bounds become numbers

**Components:** `Deconstruct Domain` ×2 (Maths → Domain), `Subtraction` ×2 (Maths → Operators)

An interval is a range, and you can't divide a range by 300. So `Deconstruct Domain` splits each one into **S** (start) and **E** (end), and `Subtraction` takes E minus S to get an ordinary length:

```text
X: 960.204105 − (−960.204105)   = 1920.408
Y: 1075.194692 − (−1075.194692) = 2150.389
```

Small step, but it's the hand-off point where geometry turns into arithmetic.

---

## 3. How many bays fit (the fencepost problem)

**Components:** `Integer Division` ×2 (Maths → Operators, A//B), `Addition` ×2 (Maths → Operators, B = 2)

`Integer Division` asks how many whole 300 cm bays fit in each extent, and it rounds down:

```text
X: 1920.408 // 300 = 6.40 → 6        Y: 2150.389 // 300 = 7.17 → 7
```

Six bays is not seven columns, though, and it's certainly not enough to reach the far edge. This is the old fencepost problem: a fence with N gaps needs N + 1 posts. Here we need one more on top of that, because the rounding-down left a partial bay hanging off the end.

A small example makes it obvious. Take a width of 10 and a spacing of 3:

```text
0 ---- 3 ---- 6 ---- 9 ---- 12
|--1--||--2--||--3--|
```

`floor(10/3)` gives 3 steps, and three steps only get you to 9. The edge at 10 is still uncovered. The points you actually need are 0, 3, 6, 9, 12, which is five. That's the 3 steps plus **2**: one because the division rounded down, one because the starting point counts as a point too.

Result: **8 points in X, 9 points in Y.**

**Is +2 safe in general?** For coverage, yes, and the algebra is short. Let q = floor(W / s). The count is then q + 2, which puts the last point (q + 1) · s from the start. Since floor(x) + 1 is always greater than x, (q + 1) · s is always greater than W. The last point lands strictly past the far edge, for any positive spacing.

Two caveats. When the extent divides exactly by the spacing (1800 and 300, say), the formula gives one more overshoot column than the minimum needs. That's harmless, because §8 culls it. And the spacing has to stay above zero, otherwise `Integer Division` divides by zero and that branch dies, so both Bay Spacing sliders carry a minimum of 50 cm.

*A side note: I would normally reach for a Ceiling component here, but this build of Grasshopper has none under Maths → Util. Integer Division + 2 does the same job and shows its reasoning on the canvas, so I kept it.*

---

## 4. Generating the coordinates

**Component:** `Series` ×2 (Sets → Sequence)

`Series` takes a start, a step and a count, and writes out a list of numbers. The input labels are confusing, and they differ between Grasshopper versions, so here is what they mean **as labeled in this build**:

| Input | Meaning | Wired to |
| --- | --- | --- |
| **S** | start | the matching Deconstruct Domain's **S** output (the west or south edge, exactly) |
| **N** | step | that axis's Bay Spacing slider (300) |
| **C** | count | the Addition result (8 or 9) |

Each axis has its own Bay Spacing slider (X and Y, both set to 300), and each one feeds its axis twice: once in §3 to work out *how many* bays fit and once here to decide *how far* each step travels. That's on purpose. Change X to 450 and the X count and the X step update together, with no way for them to drift apart. Having two sliders also means the bays don't have to be square.

Verified output:

```text
X:  −960.20, −660.20, −360.20, −60.20, 239.80, 539.80, 839.80, 1139.80          (8 values)
Y:  −1075.19, −775.19, −475.19, −175.19, 124.81, 424.81, 724.81, 1024.81, 1324.81   (9 values)
```

Both lists run past the far edge (960 and 1075). That overshoot is deliberate. The extra columns are cheap to generate and easy to remove, which is exactly what §7 and §8 do.

---

## 5. Every X meets every Y

**Component:** `Cross Reference` (Sets → List)

Feed two lists into most Grasshopper components and they get paired index by index: first with first, second with second. Do that with our X and Y lists and you get a diagonal line of points. Not a grid.

`Cross Reference` does the Cartesian product instead, pairing every X with every Y. Two synchronized output lists, 8 × 9 = **72 entries** each. If your "grid" ever comes out as a slanted row of dots, this is the component you skipped.

---

## 6. From coordinates to placed points

**Components:** `Construct Point` (Vector → Point), `Move` (Transform → Euclidean), `Deconstruct Plane` (Vector → Plane)

`Construct Point` turns the two Cross Reference lists into 72 points. But it builds them in **world coordinates**, so they all pile up around the world origin, nowhere near the building.

The reason is simple. Our numbers were measured from the rectangle's center, which makes them *local*. A local coordinate only means something once you say what it's local *to*. So `Deconstruct Plane` pulls out the rectangle's own origin (**O**), and that point goes into `Move` as the translation vector. `Move` (G = points, T = plane origin) then slides all 72 points onto the footprint.

This is the standard cure for the classic beginner symptom of a grid that generated fine and is sitting at the origin. Local coordinates plus their frame's origin give world placement.

---

## 7. Which columns are inside

**Component:** `Point In Curve` (Curve → Analysis)

| Input | Accepts | Wired to |
| --- | --- | --- |
| **C** | a curve | the `Crv` footprint rectangle |
| **P** | points | the moved grid points |

The output is one True/False per point: True means inside the boundary, False means outside. Nothing has been removed yet. It's only a list of answers, and the culling happens in the next step.

Wiring discipline matters here. C takes only a curve and P takes only points, and swapping them is the single most common mistake with this component. When it happens the component turns red, and the hover tooltip spells out the exact conversion error. Read the tooltip before you start rewiring.

One boundary behavior to know about: a point sitting *exactly on* the curve counts as inside. That matters in this model, because §4 starts each series at the domain start. The first grid lines therefore coincide with the west and south edges to the millimeter, and those columns survive.

---

## 8. Keeping the survivors

**Component:** `Cull Pattern` (Sets → List)

- **L** ← the moved points (72)
- **P** ← the True/False pattern from Point In Curve (72)

`Cull Pattern` walks both lists in lockstep. It never checks which point a True/False belongs to; it trusts the order. A point and its True/False are linked only by list position, so both lists have to come from the *same* point set. Feed it a pattern computed from a different list and it will happily delete the wrong columns without a single warning.

**Result: 72 candidates → 56 columns (7 × 8).** The 16 removed columns are the overshoot past the east and north edges.

---

## 9. What the grid told us

**The edge condition is asymmetric.** The series is anchored at the west and south edges, and 1920.408 does not divide evenly by 300, so the leftover has to land somewhere. It lands entirely on two opposite sides:

| Edge | Last column sits | Condition |
| --- | --- | --- |
| West | 0 cm from the edge | flush on the boundary |
| South | 0 cm from the edge | flush on the boundary |
| East | about 120 cm short | gap |
| North | about 50 cm short | gap |

Anchor the series at the center instead and the irregularity splits across all four sides. Which one is better is an architectural question, and it's not for the definition to answer. Flush pilotis on two faces and a centered, breathing colonnade are very different buildings. What the model can do is put that decision in front of you instead of burying it.

**Columns on the façade line are legitimate.** The boundary columns from §7 are kept by design, so don't read them as a bug.

**None of this is a structural claim.** Bay size, grid topology and column count are geometric facts. Member sizing, connections and lateral stability sit outside what this model covers (see §10).

---

## 10. Assumptions and known limits

- **Axis-aligned footprint.** The definition assumes the rectangle follows world X and Y. A rotated footprint means constructing the points *in the rectangle's own plane* instead of building them in world XY and moving them. A non-rectangular footprint goes further: the extents-based counting in §2 and §3 stops making sense, so the whole front end would need rethinking.
- **No structural analysis.** Span capacity, deflection, buckling and lateral stability all depend on member sizes, material and bracing, and none of those exist in this model. The next metric on the list is **tributary area**: Voronoi cells of the column set, clipped to the footprint. It's a modest load-distribution proxy, the kind you meet in a first structures course, and I'd rather use that than pretend to more.
- **Boundary tolerance.** `Point In Curve` falls back on Rhino's document tolerance for on-boundary decisions. A column within roughly 0.01 cm of the edge could flip in or out.
- **Edge columns are spacing artifacts.** Their 0 cm offset comes from where the series is anchored and from the 300 cm spacing. It isn't design intent encoded anywhere in the model.
- **One anchoring strategy only.** The grid is edge-anchored at west and south. A worthwhile later extension is a switch between edge-anchored and centred grids (see §9). The current version doesn't need it to be publishable.

---

## Appendix A: Component inventory

| # | Component | Ribbon path | Role |
| --- | --- | --- | --- |
| 1 | Curve | Params → Geometry | footprint reference |
| 2 | Deconstruct Rectangle | Curve → Primitive | plane + X/Y intervals |
| 3 | Deconstruct Domain ×2 | Maths → Domain | interval → S / E |
| 4 | Subtraction ×2 | Maths → Operators | E − S = extent |
| 5 | Integer Division ×2 | Maths → Operators | extent // spacing |
| 6 | Addition ×2 | Maths → Operators | steps + 2 = point count |
| 7 | Series ×2 | Sets → Sequence | coordinates per axis |
| 8 | Cross Reference | Sets → List | Cartesian product |
| 9 | Construct Point | Vector → Point | 72 candidate points |
| 10 | Deconstruct Plane | Vector → Plane | rectangle center point |
| 11 | Move | Transform → Euclidean | local → world placement |
| 12 | Point In Curve | Curve → Analysis | inside test per point |
| 13 | Cull Pattern | Sets → List | keep the survivors |

Not counted above: four number sliders feed the arithmetic, Bay Spacing X and Bay Spacing Y (300 each) and two constants set to 2 for the Addition components.

## Appendix B: Checkpoint values

If your canvas disagrees with anything in this table, stop there and debug before moving on.

| Checkpoint | Expected |
| --- | --- |
| Deconstruct Rectangle X interval | −960.204105 To 960.204105 |
| Deconstruct Rectangle Y interval | −1075.194692 To 1075.194692 |
| Subtraction results | 1920.408 / 2150.389 |
| Integer Division results | 6 / 7 |
| Addition results | 8 / 9 |
| Series X | 8 values, ends 1139.795895 |
| Series Y | 9 values, ends 1324.805308 |
| Cross Reference outputs | 72 entries each (0–71) |
| Cull Pattern output | 56 points (7 × 8) |

## Appendix C: Bug journal, in the order they bit me

1. **Grid generated at the world origin.** The coordinates were local. Fixed with `Deconstruct Plane` + `Move` (§6).
2. **Fractional counts (7.40, 8.16…).** I had used regular `Division` instead of `Integer Division`, so fractions flowed into the count input.
3. **Series mis-wiring, several rounds of it.** Start, count and step kept getting swapped, partly because the S/N/C labels differ between Grasshopper versions. What finally worked was reading the component's own tooltips and checking each result in a panel. The roles in §4 are documented *as labeled in this build*.
4. **A diagonal line instead of a grid.** I had skipped `Cross Reference`, so longest-list matching paired X and Y index by index.
5. **`Point In Curve` turned red.** C and P were swapped, with the curve going into P and the points into C. The hover error message gave it away.
6. **Ghost points in the viewport.** The `Pt` and `Move` components were still previewing after the cull. Previews are drawings, not geometry. Right-click → Preview off fixed it.
7. **Count looked wrong: expected 42, got 56.** My estimate assumed the boundary columns would be culled. They sit *exactly on* the boundary, and `Point In Curve` keeps those. 56 is correct (§7 to §9).

---

*Built as part of a Villa Savoye computational-design study. The base model stays separate and the parametric definitions are derived from it. Structural performance is deliberately not claimed.*
