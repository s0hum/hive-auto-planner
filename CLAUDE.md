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
| [319-532](index.html#L319-L532) | Constants (`FIELD`, `AUTO_LIMIT`, `LANDMARKS`), the plan model + `seedModel`, Bézier/heading geometry, `solveMove`, `enumerateRoutes` |
| [534-819](index.html#L534-L819) | App state (`model`, `sim`, `sel`, `tab`), localStorage persistence, the master `render()`, then the base64 field image (`FIELD_IMG`, ~60 KB on [line 668](index.html#L668)) and the SVG field + timeline rendering |
| [821-1073](index.html#L821-L1073) | Right rail: step inspector, states pane, routes pane |
| [1075-1477](index.html#L1075-L1477) | Java code generation, code pane, pointer/keyboard interaction, bootstrap `render()` |

### The plan model

`model.steps` is a tree of step objects, each with `kind`:

- `move` — `end`, `ctrl[]` (Bézier control points), `heading` (`tangent` / `reverseTangent` / `constant` / `linear`)
- `action` — a `method` name and a duration `ms`
- `wait` — `ms`
- `branch` — a `stateId` plus `cases[]`, each `{ valueId, steps[] }`; branches nest arbitrarily

`model.states` declares the match states a branch can switch on: `type` is `bool` or `enum`, `varName` becomes the Java supplier name, and `values[]` ids become the case keys (`"true"`/`"false"`, or enum constants). Changing a state's values requires `syncBranches` to reconcile the cases of every branch that uses it.

### Two tree walks that must stay in sync

`enumerateRoutes` ([483](index.html#L483)) and `generate`'s `flow` ([1140](index.html#L1140)) both walk the step tree with the same explicit frame-stack idiom (`frames` of `{list, i}`, popping exhausted frames, recursing into the chosen case). The first threads a pose and accumulates time to produce every reachable route (capped at `MAX_ROUTES = 96`, setting `truncated`); the second threads a pose and accumulates Java statements. A change to how steps compose — a new `kind`, new nesting — has to land in both or the simulation and the exported code will disagree.

Pose threading is why order matters: a move's start pose is the previous step's end pose, so `solveMove` is called with a running pose and the same geometry gets recomputed per route. `generate` deduplicates poses by rounded coordinates and `Path` methods by `stepId + startPose`, so the same physical move reached through different branches emits one method.

### Rendering

`render()` is the only entry point for a state change: it repairs `sim` against the current states, re-enumerates routes, picks `active` (the route whose `assign` matches `sim`), redraws everything, then `save()`s to localStorage (`hiveAutoPlanner.v2`). Everything is full re-render of `innerHTML` strings — no diffing, no framework — with events handled by delegation on `document` reading `data-*` attributes (`data-sel`, `data-add`, `data-state`/`data-val`, `data-drag`). New interactive markup needs a `data-` hook, not a listener.

### Coordinates

Field inches are the SVG user units directly (`viewBox="0 0 141.5 141.5"`), matching the Pedro Pathing Visualizer: x right, y up, heading in degrees CCW from +x. SVG y grows downward, so every y written into the SVG goes through `fy()` / `poly()` ([757-758](index.html#L757-L758)) and every pointer y comes back through `fieldXY` ([1374](index.html#L1374)). Model coordinates are always in the Pedro frame; the flip lives only at those two boundaries.

### Code generation

`generate` targets the current Pedro Pathing API (`com.pedropathing.api.Paths`, `PoseFactory`) and the Ivy command library, assuming a `Constants.create(hardwareMap)` in the user's `teamcode` package. Boolean branches become `conditional(this::supplier, whenTrue(), whenFalse())`; enum branches become `match(...)` over an `EnumMap`. Each case body becomes a generated helper method, so branch names flow into Java identifiers via `camel`/`pascal`/`enumConst`, which sanitize against `JAVA_KEYWORDS`. State readers and mechanism methods are emitted as `TODO` stubs.
