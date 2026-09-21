# Hive Auto Planner

A branching autonomous planner for **FIRST Tech Challenge BIOBUZZ (2026–27)**.

The 30-second AUTO period rarely goes one way. Your HIVE may or may not have tipped;
your alliance partner may or may not have taken the lane you wanted, or scored at all.
This tool lets you draw the whole decision tree on the field, simulate each combination,
and export a [Pedro Pathing](https://pedropathing.com/) OpMode that makes those decisions
**at runtime** rather than at INIT.

**→ [Open the planner](https://s0hum.github.io/hive-auto-planner/)**

## What it does

- **Field editor** on the real BIOBUZZ field, using the Pedro Pathing Visualizer's
  coordinate system (141.5 in. square, x right, y up, heading in degrees CCW from +x).
  Drag endpoints and Bézier control points; pick tangent, reverse-tangent, constant or
  linear heading per segment. The start pose has typed x / y / heading boxes above the
  field as well as a drag handle, so you can enter a measured pose exactly.
- **Alliance mirror, matching Pedro's own.** Plan one side, get the other. The three
  modes are Pedro Pathing's three mirrors, with the same maths, so the picture and the
  generated code agree to the inch:

  | Mode | Transform | Pedro equivalent |
  | --- | --- | --- |
  | `mirrorAroundPoint` (default) | x, y → 2c − x, y; heading → heading + 180° | `PoseFactory.mirrorAroundPoint(c)` |
  | `mirrorX` | x → 2a − x, heading → 180° − heading | `PoseFactory.mirrorX(a)`, and `Pose.mirror()` |
  | `mirrorY` | y → 2a − y, heading → −heading | `PoseFactory.mirrorY(a)` |

  The centre and the axes are the field origin, (70.75, 70.75). The default turns the
  plan 180° about that origin, which is how one alliance side maps onto the other — the
  two halves of an FTC field are a rotation of each other, not a reflection, so this is
  the mode that lets a routine be played from either side. The two reflections are there
  for fields whose halves really are mirror images.

  **Mirror** in the field toolbar overlays the mirrored routine so you can check it
  against the far half; **Mirror plan** flips the live plan over for editing (press it
  again to flip back — every mode is its own inverse); and the Code tab exports the
  mirrored alliance either **through the PoseFactory** — as-planned coordinates with
  `.mirrorAroundPoint(70.75, 70.75)` chained on, the way you'd write it by hand — or
  with the flipped **coordinates baked in**.
- **Decisions as first-class steps.** A decision branches on a match state — *hive tipped*,
  *partner start*, *partner scored*, or anything you define. Decisions nest, and everything
  after one is planned separately per case.
- **State simulator.** Set what is true on the field and the active route lights up on the
  canvas while the other branches stay ghosted.
- **Runtime budget.** Every reachable combination is timed against the 30 s AUTO period
  using a trapezoidal motion profile, so you can see which branch runs you out of time.
- **Java export.** Boolean states become `Commands.conditional(...)`, enum states become
  `Commands.match(...)` with an `EnumMap`. Both read their supplier when the robot *reaches*
  them, so a mid-auto decision works the way you'd expect. Pose and `Path` methods are
  deduplicated by geometry, and the state readers and mechanism calls come out as `TODO`
  stubs for you to wire up.

## Using it

1. Set your alliance, start pose and robot dimensions (Step tab with nothing selected;
   the start pose is also typable in the toolbar above the field).
2. Define the match states your auto has to react to (States tab).
3. Build the plan in the left rail: **Move**, **Action**, **Wait**, **Decision**.
4. Flip the state chips to walk each branch; check the Routes tab for the slowest one.
5. Copy the OpMode from the Code tab into your `teamcode` package — once as planned, and
   once from the mirrored side if you want the matching OpMode for the other alliance.
   The mirrored copy differs from the first by one line when you export it through the
   PoseFactory, so it stays easy to diff as the plan changes.

Plans are kept in your browser's local storage, and the JSON view in the Code tab lets you
copy a plan out or paste one back in to move it between machines.

## Things to check before you trust it

- Field landmark coordinates (CELL centers, FLOWERS, LOADING ZONES, GARDENS) are measured
  off the field image, not the official CAD. Verify them to the inch before relying on them.
- **The mirror is only as symmetric as the field really is.** The LOADING ZONES, GARDENS
  and FLOWERS in this tool pair up cleanly under the default `mirrorAroundPoint`, but the
  CELL centers as measured do not (about 2.5 in. off), so treat a mirrored plan as a
  starting point and re-check every pose that has to line up on a CELL. Note also that
  Pedro's own `Pose.mirror()` is the left-right `mirrorX` reflection, not this rotation —
  it suits fields whose halves are mirror images, and it will put a BIOBUZZ plan in the
  wrong corner.
- Timings come from a motion profile, not your robot. Calibrate the velocity and
  acceleration numbers against a real run.
- The generated code targets the current Pedro Pathing API (`com.pedropathing.api.Paths`,
  `PoseFactory`) and the Ivy command library. It assumes a `Constants.create(hardwareMap)`
  in your `teamcode` package, the same as the official quickstart.

## Credits

Built on the coordinate system, field assets and code-generation conventions of the
[Pedro Pathing Visualizer](https://github.com/Pedro-Pathing/Visualizer), and on
[Pedro Pathing](https://github.com/Pedro-Pathing/PedroPathing) and
[Ivy](https://github.com/Pedro-Pathing/Ivy). Game rules and field dimensions are from the
*BIOBUZZ Competition Manual V1*.

FIRST, FIRST Tech Challenge and BIOBUZZ are trademarks of FIRST. This is an independent
team tool and is not affiliated with or endorsed by FIRST.
