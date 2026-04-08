---
name: Review Training
description: Review an athlete's recent training history, assess load and progression, and adjust upcoming workouts using the freediver.club MCP tools. Supports both athlete and coach views.
---

## Overview

Use this skill when the user asks to review recent training, assess progress, check if the plan is on track, or adjust upcoming sessions based on what has been completed.

## When to Apply

- User asks to "review my training", "how is my plan going", or "adjust my workouts"
- User mentions missing sessions, feeling overtrained, or wanting to recalibrate
- After completing a mesocycle block and preparing for the next phase
- Coach wants to review athlete progress across a training center

## Athlete Workflow

### 1. Pull recent data

**REQUIRED — call user-profile first:**

```
Call user-profile → confirm timezone and pace speeds are current
Call getWorkouts(start, end) → last 2–4 weeks (max 90-day range per call)
```

Use the returned `timezone` to build `start` and `end` as full ISO 8601 datetimes with the correct UTC offset (e.g. `"2025-05-01T00:00:00+02:00"`). Never pass a date-only string.

Look for:
- Sessions completed vs. planned
- RPE values (high RPE on easy sessions = accumulated fatigue)
- Skipped or shortened sessions
- Progression trend in distances, depths, or hold times

### 2. Assess training load

Signs the plan is **too aggressive**:
- Multiple skipped sessions
- RPE consistently 8–10 on moderate-intensity sessions
- Athlete reports fatigue, soreness, or poor sleep

Signs the plan is **too easy**:
- RPE consistently 3–4 on prescribed moderate sessions
- Athlete completing all sessions comfortably with energy to spare
- No progression in performance metrics

### 3. Apply corrections

**If overloaded**: trigger a deload week immediately — reduce volume 40–50 %, drop intensity one notch (normal → low pace). Use `updateWorkout` to modify upcoming sessions.

**If progressing well**: continue current phase. If 3+ weeks of continuous loading, schedule the next deload.

**If underloaded**: increase one variable only — either volume (+10–15 %) OR intensity (one pace step or +2–3 m depth). Never both.

### 4. Update upcoming workouts

Use `updateWorkout` with only `workoutUuid` and the fields to change. Do not pass `trainingCenterUuid` or `connectionId` when updating.

## Coach Workflow

### 1. Get training center context

```
Call getTrainingCenters → list centers and consenting athletes
```

### 2. Review athlete workouts

```
Call getAthleteWorkouts(trainingCenterUuid, connectionId, start, end)
```

Build `start` and `end` using the athlete's timezone from `user-profile` — full ISO 8601 datetimes with UTC offset. Never pass date-only strings.

Review the same signals as the athlete workflow above, but for each athlete independently.

### 3. Create or update athlete workouts

- **Creating**: pass `trainingCenterUuid` + `connectionId` to `createWorkout` (both required)
- **Updating**: call `updateWorkout` with only `workoutUuid` and changed fields — do NOT pass `trainingCenterUuid` or `connectionId`

## Periodization Checkpoints

| After week | Action |
|---|---|
| Week 3 or 4 | Deload if not already scheduled |
| End of mesocycle (4–6 weeks) | Assess phase goal achievement, transition to next phase |
| 10 days before competition | Begin taper — cut volume 50–70 %, keep 1–2 quality sessions |
| Post-competition | Schedule recovery/transition week (very low volume and intensity) |

## Progression Limits (per week)

- Pool distance: max +10–15 %
- Static hold times: +10–15 s (intermediate), +5–10 s (beginner)
- Depth: +2–3 m (intermediate), +1–2 m (beginner)
- Never increase both volume and intensity in the same week

## API Limits

`getWorkouts` and `getAthleteWorkouts` cap at **90 days per call**. For longer history, split into multiple calls.
