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
  linear heading per segment.
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

1. Set your alliance, start pose and robot dimensions (Step tab, with nothing selected).
2. Define the match states your auto has to react to (States tab).
3. Build the plan in the left rail: **Move**, **Action**, **Wait**, **Decision**.
4. Flip the state chips to walk each branch; check the Routes tab for the slowest one.
5. Copy the OpMode from the Code tab into your `teamcode` package.

Plans are kept in your browser's local storage, and the JSON view in the Code tab lets you
copy a plan out or paste one back in to move it between machines.

## Things to check before you trust it

- Field landmark coordinates (CELL centers, FLOWERS, LOADING ZONES, GARDENS) are measured
  off the field image, not the official CAD. Verify them to the inch before relying on them.
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
