# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A branching autonomous-path planner for the FIRST Tech Challenge BIOBUZZ (2026–27) game. It draws a decision tree of robot moves on the field, simulates every branch combination against the 30 s AUTO limit, and exports a Pedro Pathing + Ivy Java OpMode that reads the match state **at runtime** rather than at INIT. See [README.md](README.md) for user-facing behavior and the caveats about landmark coordinates and timing accuracy.

## Build / run / test

There is none of any of these. The entire app is [index.html](index.html) — one static file, no build step, no dependencies, no test suite, no package.json. To work on it, open the file in a browser (`start index.html`) and reload. It is published straight from `main` to GitHub Pages.

Since there are no tests, verify changes by loading the page: the seeded example plan (`seedModel`) exercises boolean branches, enum branches, moves, actions and the code generator, so a smoke test is "load the page, check the field renders, flip the state chips, open the Code tab."

## Architecture

`index.html` is four sequential `<script>` blocks that share one global scope and run in load order. There are no modules or imports; every function and `let` is a global, and later blocks call into earlier ones. Preserve that ordering when adding code.

| Lines | Role |
| --- | --- |
| [362-892](index.html#L362-L892) | Constants (`FIELD`, `AUTO_LIMIT`, `LANDMARKS`), the mecanum drivetrain model, the plan model + `seedModel`, the action-library resolvers, Bézier/heading geometry, `solveMove`, `enumerateRoutes` |
| [894-1385](index.html#L894-L1385) | App state (`model`, `sim`, `sel`, `tab`), localStorage persistence + `migrate`, the master `render()`, then the base64 field image (`FIELD_IMG`, ~60 KB on [line 1174](index.html#L1174)) and the SVG field + timeline rendering |
| [1387-1865](index.html#L1387-L1865) | Right rail: step inspector, states pane, actions pane, routes pane |
| [1867-2429](index.html#L1867-L2429) | Java code generation, code pane, pointer/keyboard interaction, bootstrap `render()` |

### The plan model

`model.steps` is a tree of step objects, each with `kind`:

- `move` — `end`, `ctrl[]` (Bézier control points), `heading` (`tangent` / `reverseTangent` / `constant` / `linear`). A `linear` heading stores only `endDeg`: it interpolates from the pose the move is launched from, so heading carries step to step. `headingAt`/`headingAtIndex`/`traverseTime` therefore take that incoming angle as an `h0` argument, and `solveMove` passes `pose.h` for it.
- `action` — an `actionId` pointing into `model.actions`, and nothing else. The name,
  Java `method`, duration `ms` and `note` all live on the library entry, so `actionOf`
  ([535](index.html#L535)) is the only place that resolves the pointer — it takes the
  plan explicitly because the mirrored copy and `generate` each carry their own
  `actions`. `stepMs` and `stepLabel` sit beside it and are what the route timer, the
  tree, the timeline and the code generator read instead of touching the step.
- `wait` — `ms`
- `branch` — a `stateId` plus `cases[]`, each `{ valueId, steps[] }`; branches nest arbitrarily
- `parallel` — a `mode` (`all` / `deadline` / `race`, the `PARALLEL_MODES` table, which also
  carries the Ivy `Groups` method each one exports as) plus `legs[]`, each
  `{ id, name, steps[] }`. A leg is a **flat** sequence of `move` / `action` / `wait`: the
  tree offers no container buttons inside one, which is what keeps `solveLeg` and the two
  tree walks from having to fork a decision that only exists on one of several concurrent
  clocks. `solveParallel` ([803](index.html#L803)) is the only place group timing lives —
  `all` takes the slowest leg, `race` the fastest, `deadline` leg 1 — and it picks the one
  driving leg whose end pose becomes the group's. `extraDrive` counts legs that also drive
  (a plan error: one follower, two commands) and `clipped` says the group ends before the
  driving leg's path does, so the pose it threads on is a guess.

`childLists(s)` ([571](index.html#L571)) is the single answer to "what step lists hang off
this step" — a decision's cases, a group's legs. Every walk that only cares about
containment (`findStep`, `countSteps`, `reid`, `actionUsage`, `syncBranches`, the migrate
passes) goes through it, so a third container kind lands in all of them at once. The walks
that thread a pose — `enumerateRoutes`, `generate`'s `flow`, `solveLeg` — do not, because
each has to know what the nesting *means*.

`model.actions` is the library an action step points at — `{ id, name, method, ms, note }`
per entry, edited in the Actions tab. Two steps that share an entry are one Java method and
one duration; deleting an entry is blocked while any step still uses it (`actionUsage`), so
an `actionId` normally resolves. It can still dangle in hand-edited JSON, and every reader
falls back rather than throwing.

`model.states` declares the match states a branch can switch on: `type` is `bool` or `enum`, `varName` becomes the Java supplier name, and `values[]` ids become the case keys (`"true"`/`"false"`, or enum constants). Changing a state's values requires `syncBranches` to reconcile the cases of every branch that uses it.

### Two tree walks that must stay in sync

`enumerateRoutes` ([822](index.html#L822)) and `generate`'s `flow` ([1962](index.html#L1962)) both walk the step tree with the same explicit frame-stack idiom (`frames` of `{list, i}`, popping exhausted frames, recursing into the chosen case). The first threads a pose and accumulates time to produce every reachable route (capped at `MAX_ROUTES = 96`, setting `truncated`); the second threads a pose and accumulates Java statements. A change to how steps compose — a new `kind`, new nesting — has to land in both or the simulation and the exported code will disagree. A `parallel` step is one frame-stack entry in both: each defers to `solveParallel` for the group's end pose, so the legs never enter the frame stack.

A route carries a group as a **single** item of `kind:"parallel"` whose `time` is the
group's elapsed time, not the sum of its legs — which is why anything looking for the
moves a route drives calls `moveItems(route)` ([884](index.html#L884)) rather than
filtering `route.items`. The field drawing, the drag handles and `liveIds` all go through
it; the timeline and `poseAtFrac` deliberately do not, because to them a group is one
block on the clock.

Pose threading is why order matters: a move's start pose is the previous step's end pose, so `solveMove` is called with a running pose and the same geometry gets recomputed per route. `generate` deduplicates poses by rounded coordinates and `Path` methods by `stepId + startPose`, so the same physical move reached through different branches emits one method.

### The drivetrain model

`model.robot` is a description of a mecanum base — `motorRpm`, `wheelDia`, `gearRatio`, `massLb`, `eff`, plus the `w`/`h` used for drawing — not a set of motion limits. `drivetrain()` ([426](index.html#L426)) turns it into the numbers the timing needs, and nothing outside that function should read the raw fields to compute speed.

Timing is therefore direction-dependent, which is the point: the four wheels share one speed budget (`holoCap`), so travelling sideways or turning while translating both cost from it. `traverseTime` ([644](index.html#L644)) samples the curve, caps the speed at every sample, then fits a profile under the caps with a forward pass limited by the motor's torque curve and a backward pass limited by braking. That replaced a scalar trapezoid, so a move's time no longer depends only on its length — the heading mode changes it too.

Acceleration is derived, not entered: `MOTOR_NM_RPM` encodes that stall torque x free speed is near constant across a goBILDA 5203's gearbox range, which is what lets one RPM figure stand in for a torque spec. `MU` caps it at the tiles' grip.

Old plans carry `maxVel`/`maxAccel`/`maxDecel`/`angVel`; `migrate` drops them at v3 and backfills `DRIVE_DEFAULTS`, and v4 strips every linear heading's `startDeg`. v5 lifts each action step's `name`/`method`/`ms` into `model.actions`, keyed on `method|ms` so two steps that called one method for different lengths stay two entries rather than silently retiming one. It runs on pasted-in JSON as well as localStorage.

### Rendering

`render()` is the only entry point for a state change: it repairs `sim` against the current states, re-enumerates routes, picks `active` (the route whose `assign` matches `sim`), redraws everything, then `save()`s to localStorage (`hiveAutoPlanner.v2`). Everything is full re-render of `innerHTML` strings — no diffing, no framework — with events handled by delegation on `document` reading `data-*` attributes (`data-sel`, `data-add`, `data-state`/`data-val`, `data-drag`). New interactive markup needs a `data-` hook, not a listener.

### Coordinates

Field inches are the SVG user units directly (`viewBox="0 0 141.5 141.5"`), matching the Pedro Pathing Visualizer: x right, y up, heading in degrees CCW from +x. SVG y grows downward, so every y written into the SVG goes through `fy()` / `poly()` ([1297-1298](index.html#L1297-L1298)) and every pointer y comes back through `fieldXY` ([2237](index.html#L2237)). Model coordinates are always in the Pedro frame; the flip lives only at those two boundaries.

### Code generation

`generate` targets the current Pedro Pathing API (`com.pedropathing.api.Paths`, `PoseFactory`) and the Ivy command library, assuming a `Constants.create(hardwareMap)` in the user's `teamcode` package. Boolean branches become `conditional(this::supplier, whenTrue(), whenFalse())`; enum branches become `match(...)` over an `EnumMap`. Each case body becomes a generated helper method, so branch names flow into Java identifiers via `camel`/`pascal`/`enumConst`, which sanitize against `JAVA_KEYWORDS`. A parallel group becomes `parallel(...)` / `deadline(...)` / `race(...)` with one argument per leg, in editor order — which is what makes `deadline` mean anything, since Ivy reads its first argument as the deadline. A leg of one statement goes inline; a longer one becomes a helper, deduplicated by its body in `legHelper` (safe in a way `addHelper` is not: a leg carries no continuation, so two legs that read alike really are one method). State readers and mechanism methods are emitted as `TODO` stubs — one stub per distinct
`camel(method)`, so two library entries spelling the same method collapse into one.
