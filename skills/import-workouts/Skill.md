---
name: import-workouts
description: Batch import historical training data from CSV, spreadsheet, or free-text logs into freediver.club. Handles Polish/English abbreviations, execution-vs-plan reconciliation, timed swims, dive simulations, competitions, and complex set structures.
---

## Overview

Use this skill to backfill past training sessions from external sources (CSV exports, Google Sheets, handwritten logs, coach notes) into the freediver.club workout system via MCP.

## When to Apply

- User provides a file (CSV, text, JSON) with historical training data
- User asks to "import workouts", "backfill training", or "upload my training log"
- User pastes training data from a spreadsheet or document

## Workflow

### 1. Read source data and user profile

```
Call user-profile → timezone (for date-time offset) and disciplineSpeeds
Read the source file/data
```

Determine timezone offset per date:
- Check DST transitions for the user's IANA timezone
- Europe/Warsaw: CET (+01:00) Oct→Mar, CEST (+02:00) Mar→Oct
- Apply correct offset to each workout's `scheduled_at`

### 2. Parse and interpret training entries

Map common abbreviations to disciplines:

| Abbreviation | Discipline |
|---|---|
| DYN, mono | `dynamic_apnea` |
| DYNB, DYNBF, DBF, BF, bifins | `dynamic_apnea_with_bifins` |
| DNF, no-fins | `dynamic_apnea_no_fins` |
| STA, static | `static_apnea` |
| CWT, mono depth | `constant_weight` |
| CWTB, CWT-BF | `constant_weight_with_bifins` |
| CNF | `constant_weight_no_fins` |
| FIM | `free_immersion` |

### 3. Ask clarifying questions

Before converting, compile a list of ambiguities:
- Entries without discipline labels → ask user
- Unclear rest schemes → confirm format
- Dive simulation parameters (freefall depth, descent/ascent rates)
- Sessions to skip vs include (empty, equipment-only)
- Default session time if not specified
- Competition handling preferences

Batch questions efficiently (max 4 at a time). Make reasonable assumptions for clear cases; only ask about genuinely ambiguous entries.

### 4. Convert to workout JSON

Write a JSON file with all workouts for user review before uploading.

### 5. Get approval and create via MCP

After user approves, call `createWorkout` for each entry sequentially.

---

## Conversion Rules

### Backfill fields

For historical/completed workouts, set:
- `completed: true`
- `notes` — athlete's post-session observations (from description/notes column)
- `rpe` — only when explicitly stated (number or "X/10" format)

### Execution overrides plan

**Critical rule**: When notes describe execution different from the plan, record the ACTUAL execution:
- "125m podzielone na 100m i 25m" → model as 100m + 25m (not 125m)
- "do 55m, później kilka wdechów" → distance = 55m (not planned 100m)
- "przerwane" / "nie dokończone" → remove unfinished sets
- "ostatnia próba 5:20" → use 320s for that rep (not planned 240s)
- Split/broken attempts → record the continuous breath-hold distance achieved

When someone achieves MORE than planned, record the actual higher value.

### Time-based vs distance-based

Use `time` field (seconds) on pool discipline elements when:
- Plan specifies swim duration, not distance ("3:30 DYN", "swim for 2:00")
- Timed intervals ("4x30s, 4x60s, 4x90s")
- Goal is apnea time under movement, not covering distance

Use `distance` field when:
- Plan specifies meters ("4x100m", "150m")
- Pyramid sets with specific distances

Never set both `time` and `distance` on the same element.

### Rest schemes

| Format | Interpretation |
|---|---|
| "przerwa 45sek/25m" | Rest scales: `(distance/25) × 45` seconds |
| "przerwa 30sek/25m" | Rest scales: `(distance/25) × 30` seconds |
| "R5:00, R4:00" | Explicit rest per set (use break_time) |
| "2x czas płynięcia" | `dynamic_rest_option: "twice_dive_time"` |
| "3x czas" | `dynamic_rest_option: "three_dive_time"` |
| "jeden wdech" | `rest_type: "free_text"`, `free_text_label: "Jeden wdech"` |
| "X min przerwy" | `fixed_rest_time: X*60` |

### Pyramid sets with scaling rest

When rest scales with distance (e.g., 30s per 25m):
- Each distance tier becomes its own drill
- `fixed_rest_time` = `ceil(distance/25) × rest_per_25m` (or `floor` depending on convention)
- `break_time` = same as rest (unless transitioning between phases)

### STA + DYN combo sets

Pattern: "2:30 STA + 100m DYNB"

Model as single drill, 1 rep, 2 elements:
```json
{
  "reps": 1,
  "rest_type": null,
  "break_time": 300,
  "elements": [
    { "discipline": "static_apnea", "time": 150, "pace": "normal", "static_apnea_goal": 4 },
    { "discipline": "dynamic_apnea_with_bifins", "distance": 100, "pace": "normal" }
  ]
}
```

When multiple sets with different STA durations: each becomes its own drill.

### Competitions

- `completed: true`
- `pace: "high"` on the competition element
- Results (distance achieved, split times) go in `notes`
- `rpe` if athlete provided it
- Name format: "Zawody - DISCIPLINE DISTANCEm" or "Competition - ..."
- Model as single drill, 1 rep at the achieved distance

### Dive simulations (pool-based depth simulation)

Three-phase structure per target depth:
1. **Descent (DYN)**: swim simulating active descent to freefall start
2. **Freefall (STA)**: static hold simulating freefall + bottom time
3. **Ascent (DYN)**: swim simulating ascent to surface

Parameters needed from user:
- `freefallStartDepth` (meters) — where active kick stops
- `descentRate` (m/s) — speed of descent
- `ascentRate` (m/s) — speed of ascent
- `bottomTime` (seconds) — time at depth before turning (default: 5s)

Calculations:
```
initialDynamicTime = freefallStartDepth / descentRate
staticTime = (targetDepth - freefallStartDepth) / descentRate + bottomTime
ascentTime = targetDepth / ascentRate
```

Convert swim times to distance: `distance = time × user's low pace speed`
Round to nearest 25m for pool constraints (or keep exact if non-standard pool).

Model as drill per depth:
```json
{
  "name": "70m simulation",
  "reps": 1,
  "rest_type": null,
  "break_time": 300,
  "elements": [
    { "discipline": "dynamic_apnea_with_bifins", "distance": 25, "pace": "low", "notes": "Descent ~31s" },
    { "discipline": "static_apnea", "time": 61, "pace": "normal", "static_apnea_goal": 4, "notes": "Freefall + bottom" },
    { "discipline": "dynamic_apnea_with_bifins", "distance": 75, "pace": "low", "notes": "Ascent ~70s" }
  ]
}
```

### Contractions training

| Pattern | Model |
|---|---|
| "STA X:XX na skurczach" | Timed hold with `static_apnea_goal: 4`, time = target seconds |
| "do pierwszego skurcza" | Open-ended hold with `static_apnea_goal: 1` (no time field) |
| "Xmin skurcze - Ymin przerwy" | goal=4, time=X*60, fixed_rest_time=Y*60 |

### Hypoxic / minimal-rest sets

Pattern: "16x25 jeden wdech"
```json
{
  "reps": 16,
  "rest_type": "free_text",
  "free_text_label": "Jeden wdech",
  "elements": [{ "discipline": "...", "distance": 25, "pace": "normal" }]
}
```

### Nested set structures

Pattern: "((50m slow, 50m fast) x4 - 30s break) x6 - 2m break"

Model as N identical drills (one per outer rep), each with inner reps:
```json
{
  "name": "Set 1",
  "reps": 4,
  "rest_type": "fixed",
  "fixed_rest_time": 30,
  "break_time": 120,
  "notes": "Alternate slow/fast each 50m",
  "elements": [{ "discipline": "...", "distance": 50, "pace": "normal" }]
}
// ... repeated for each outer set
```

If pace alternates within reps and can't be expressed in elements, add notes describing the pattern.

### Non-standard pool lengths

When distances don't align with 25m multiples (e.g., 30, 55, 80, 110):
- Keep exact distances as recorded by athlete
- Pool may be non-standard length (e.g., 27.5m)
- Rest scheme still applies using the stated formula

### Weighting / equipment sessions with drills

When a session combines equipment work with actual swimming:
- Include drills for the swimming portion
- Put equipment details (weights, wetsuit) in `notes`
- `workout_type: "pool-training"` (since drills exist)

### Sessions to skip

Do not create workouts for:
- Completely empty rows (no plan, no notes)
- Pure equipment sessions with no swimming (unless user says otherwise)

---

## Output Format

Write to `{downloads_dir}/workouts-import.json` as a JSON array:
```json
[
  {
    "scheduled_at": "2025-01-13T07:45:00+01:00",
    "workout_type": "pool-training",
    "name": "DYN Pyramid 4x50, 3x75, 2x100",
    "completed": true,
    "notes": "...",
    "rpe": 7,
    "drills": [...]
  }
]
```

Present a summary table to user showing count by month, types, and key notes before proceeding with upload.

---

## Upload Execution

After approval, call `createWorkout` for each workout in chronological order. Report progress every 10 workouts. If any creation fails, log the error and continue with remaining workouts. Report failures at the end.
