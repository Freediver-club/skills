---
name: Create Workout
description: Design and create a single freediving training session using the freediver.club MCP tools. Covers pool, depth, and supporting workout types with correct drill/element structure.
---

## Overview

Use this skill when the user asks to create, design, or add a single training session. This skill covers the full workout data model, pace-based time estimation, workout recipes, and the `createWorkout` API call.

## When to Apply

- User asks to "create a workout", "add a session", or "design a pool/depth/strength session"
- User wants a specific workout type (CO2 table, O2 table, depth progression, etc.)
- A planning skill needs to produce individual workout objects

## Before Building Any Workout

```
Call user-profile → get disciplineSpeeds and timezone
```

Use `disciplineSpeeds` to estimate dive/swim duration: `time = distance / speed` (pool) or `time = 2 × depth / speed` (depth). Verify rest intervals are safe before creating.

Default speeds when no custom values exist:

| Discipline | Low (m/s) | Normal (m/s) | High (m/s) |
|---|---|---|---|
| DYN / DYNB | 0.71 | 1.25 | 1.47 |
| DNF | 0.71 | 0.89 | 1.00 |
| CWT / CWTB | 0.63 | 0.83 | 1.25 |
| FIM | 0.56 | 0.71 | 1.00 |
| CNF | 0.50 | 0.63 | 0.83 |

## Workout Structure

```
Workout
├── scheduled_at       (ISO 8601 date)
├── workout_type       (pool-training | depth-training | stretching | strength | other | sauna)
├── name               (short descriptive title)
└── drills[]           (omit for supporting sessions)
    ├── name           (block label: "Warm-up", "Main set", etc.)
    ├── reps
    ├── rest_type      (fixed | dynamic | free_text | null)
    ├── fixed_rest_time / dynamic_rest_option / free_text_label
    ├── break_time     (seconds after this drill, before next)
    ├── notes
    └── elements[]
        ├── discipline
        ├── pace       (low | normal | high)
        ├── distance   (pool disciplines, meters)
        ├── depth      (depth disciplines, meters, max 150)
        ├── time       (static apnea / time-based holds, seconds)
        ├── static_apnea_goal  (1 = contractions, 4 = timed hold)
        └── notes
```

### Key structural rules

- One drill = one training block (warm-up, main set, cool-down are separate drills)
- Elements are sequenced within a rep; rest is between reps
- `break_time` on a drill = rest before the next drill (match to that drill's rep rest by default)
- Do NOT set `break_time` on the last drill
- Do NOT set `rpe`, `notes` (workout-level), or `completed` on planned workouts

## Recovery Ratios

| Type | Minimum rest |
|---|---|
| Pool dynamics | ≥ 1× swim time (beginners: 1.5–2×) |
| Static apnea | ≥ 2× hold time (beginners: 2–3×) |
| Depth dives | ≥ 2× dive time (deeper dives: 3×) |

## Session Structure

- **Warm-up**: 2–4 × 25–50 m at low pace (pool) or 1–3 easy dives to 30–50 % of target depth
- **Main set**: core stimulus — verify total duration is realistic
- **Cool-down**: 1–2 easy short efforts or surface swimming (`other` discipline)

## Workout Recipes

### Pool — CO2 table

Goal: build CO2 tolerance through progressively shorter rest. Model as multiple single-rep drills with decreasing `break_time`, or use `free_text_label` to note the descending pattern.

```
Drill: "CO2 Table"  reps: 1 per step
  elements: [{ discipline: "dynamic_apnea_with_bifins", distance: 50, pace: "normal" }]
  break_time: 120, 105, 90, 75, 60, 45, 30  (one drill per rest step)
```

### Pool — O2 table

Goal: stress O2 stores through progressively longer swims with constant rest.

```
Multiple single-rep drills with increasing distance, constant break_time: 120
  distances: 25, 37, 50, 62, 75, 87, 100, 112 m
```

### Pool — interval training

```
Drill: "Main set"
  reps: 6, rest_type: "fixed", fixed_rest_time: 90, break_time: 180
  elements: [{ discipline: "dynamic_apnea_with_bifins", distance: 75, pace: "normal" }]
```

### Static apnea — progressive holds

```
Drill: "Static holds"
  reps: 5, rest_type: "fixed", fixed_rest_time: 120
  elements: [{ discipline: "static_apnea", time: 120, pace: "normal", static_apnea_goal: 4 }]
```

### Static apnea — contractions (advanced only)

```
Drill: "Contractions set"
  reps: 3, rest_type: "fixed", fixed_rest_time: 180
  elements:
    - { discipline: "static_apnea", pace: "normal", static_apnea_goal: 1 }
    - { discipline: "static_apnea", time: 30, pace: "normal", static_apnea_goal: 4 }
```

Contractions element must be first. Max two elements per drill when using contractions. Only prescribe for experienced athletes or when coach explicitly requests.

### Depth — progressive dives

```
Drill: "Warm-up dives"
  reps: 2, rest_type: "dynamic", dynamic_rest_option: "three_dive_time", break_time: 300
  elements: [{ discipline: "free_immersion", depth: 15, pace: "low" }]

Drill: "Working dives"
  reps: 4, rest_type: "dynamic", dynamic_rest_option: "three_dive_time"
  elements: [{ discipline: "constant_weight_with_bifins", depth: 30, pace: "normal" }]
```

### Depth — hang training

```
Drill: "Hangs"
  reps: 3, rest_type: "dynamic", dynamic_rest_option: "twice_dive_time"
  elements:
    - { discipline: "free_immersion", depth: 10, pace: "low", notes: "Descend to 10m, hang 60s, ascend" }
    - { discipline: "hang", time: 60, pace: "low" }
```

### Supporting sessions (no drills needed)

```json
{ "scheduled_at": "2025-06-10", "workout_type": "strength",   "name": "Lower body + core" }
{ "scheduled_at": "2025-06-11", "workout_type": "stretching", "name": "Diaphragm + hip mobility" }
{ "scheduled_at": "2025-06-14", "workout_type": "sauna",      "name": "Sauna — 3 × 12 min" }
{ "scheduled_at": "2025-06-12", "workout_type": "other",      "name": "Active recovery — breathing exercises" }
```

## Common Mistakes to Avoid

| Mistake | Correct approach |
|---|---|
| Setting `rpe` or `completed` on planned workouts | Leave unset — athlete fills these post-session |
| `rest_type: null` on apnea drills | Always prescribe rest between apnea reps |
| Skipping warm-up or cool-down | Every water session must start and end with easy efforts |
| Depth plan without warm-up dives | Always start with 2–3 easy dives at 30–50 % working depth |
| Setting `break_time` on the last drill | Only set between drills, not after the last one |
