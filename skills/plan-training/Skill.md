---
name: plan-training
description: Create a periodized multi-week freediving training plan for an athlete using the freediver.club MCP tools. Covers macrocycle design, weekly templates, and workout creation.
---

## Overview

Use this skill when the user asks to create a training plan, design a program, or schedule a block of freediving training. This skill guides the full planning workflow from athlete profile through workout creation.

## When to Apply

- User asks to "create a training plan", "design a program", or "plan my next 4/8/12 weeks"
- User mentions a competition date and wants to peak for it
- User wants to start a new training cycle (base, build, or peak phase)

## Step-by-Step Workflow

### 1. Gather context

**REQUIRED — call user-profile before any other tool:**

```
Call user-profile → get disciplineSpeeds and timezone
Call getWorkouts (last 4–8 weeks) → understand current load, frequency, progression
```

Use the returned `timezone` to construct all date range arguments and `scheduled_at` values as full ISO 8601 datetimes with the correct UTC offset (e.g. `"2025-06-10T09:00:00+02:00"`). Never pass a date-only string.

Ask the athlete (or infer from history):
- Current level (beginner / intermediate / advanced)
- Primary target discipline
- Competition date (if any)
- Available training days per week
- Pool and/or depth access

### 2. Design the macrocycle

Map mesocycle phases from today to the target date (or rolling 8–12 weeks if no competition):

| Phase | Goal | Volume | Intensity |
|---|---|---|---|
| Base / Aerobic | General fitness, CO2 tolerance, technique | High | Low–Moderate |
| Build / Specific | Discipline-specific capacity | Moderate–High | Moderate–High |
| Peak / Competition | Sharpen performance, reduce fatigue | Low | High (quality) |
| Recovery / Transition | Physical and mental restoration | Very low | Very low |

Every 3rd or 4th week: deload — reduce volume 40–50 %, intensity one notch.

### 3. Build weekly templates

Distribute water sessions, strength, stretching, and rest. Minimum recovery rules:
- Never stack high-intensity sessions on consecutive days
- Place strength ≥ 24 h before a key water session
- Beginner: 2–3 water sessions/week max; Intermediate: 3–4; Advanced: 4–5

Example intermediate microcycle:

| Day | Session |
|---|---|
| Monday | Pool — technique & CO2 tables |
| Tuesday | Strength |
| Wednesday | Rest or light stretching |
| Thursday | Pool — main performance set |
| Friday | Stretching / yoga |
| Saturday | Depth session |
| Sunday | Rest or sauna |

### 4. Write individual workouts

Create each workout with `createWorkout`, starting from the nearest week.

- Use descriptive `name` values (e.g. "Pool — CO2 table + technique", "Depth — progressive CWT")
- Structure: warm-up drill → main set drill(s) → cool-down drill
- Cross-check estimated session duration against available time using `disciplineSpeeds`
- Supporting sessions (strength, stretching, sauna, other) need no drills — just `scheduled_at` (ISO 8601 datetime with time + UTC offset), `workout_type`, and `name`

### 5. Competition taper (if applicable)

7–10 days before competition:
- Reduce volume 50–70 %
- Keep 1–2 high-quality sessions
- Prioritize sleep, nutrition, mental prep
- Mark competition day with `is_competition: true` via `updateWorkout`

## Progressive Overload Rules

- **Pool distance**: max +10–15 % per week
- **Static hold times**: max +10–15 s/week (intermediate); +5–10 s (beginner)
- **Depth**: max +2–3 m/week (intermediate); +1–2 m (beginner)
- **Never increase both intensity AND volume in the same week**

## Safety Reminders

- Default to sub-maximal training (70–85 % of personal best)
- Always include adequate rest between apnea efforts
- Depth training requires a safety buddy — add a note in the workout
- Do not prescribe deeper targets unless equalization is confirmed comfortable
