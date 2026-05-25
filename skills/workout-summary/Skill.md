---
name: workout-summary
description: Guide athletes through reviewing and logging their Freediver Club workouts via structured conversation, then store meaningful notes for their coach. Use this skill whenever a user wants to discuss, review, debrief, or log a freediving training session — including phrases like "let's go through today's training", "ogarnijmy trening", "review my workout", "summarise today's session", or any reference to a completed pool, depth, or static apnea session. Also use when the user wants to compare performance across sessions or analyse trends across recent workouts.
---

# Freediver Workout Review Skill

This skill guides an athlete through a structured post-workout debrief and stores meaningful notes for their coach in the Freediver app.

## Core Philosophy

Notes must be **coach-meaningful** — focused on what the coach cannot observe: mental state, physical limiters, technique discoveries, unusual sensations, and subjective effort. Do NOT include:
- Obvious facts derivable from the plan (e.g. "completed all reps" when the plan prescribes X reps)
- Information fabricated from memory of previous turns — only store what the athlete said **in the current debrief**
- Comparisons to other discipline types (never compare DYN to DYNb — always compare like-for-like)

## Workflow

### Step 1: Load today's workout
Always call `Freediver:user-profile` first to get timezone, then `Freediver:getWorkouts` for today's date range. Identify the workout type (pool/depth/static) before asking questions.

### Step 2: Ask targeted questions
Ask the minimum questions needed to produce a meaningful note. Tailor questions to workout type:

**Pool dynamics (DYN / DYNb / DNF):**
- Main effort: distance achieved, limiter (physical / CO₂ / mental / technique), perceived surplus
- Repeating sets: CO₂ build-up pattern, when discomfort started, any hypoxia signs
- Technique: specific cues observed or broken (e.g. arm pull timing, body position, kicks)
- Mental state: relaxation, negotiation with the brain, any "stop" decisions
- RPE (1–10)

**Static apnea (STA):**
- When contractions started per rep and how they evolved across the set
- Overall ease/difficulty, mental state
- Adaptation pattern within the session
- RPE (1–10)

**Depth (CWT / FIM / CNF):**
- Target depth vs actual, any equalization issues
- Freefall quality, turn, pull-out
- Mental state at depth
- RPE (1–10)

Ask one group of questions at a time. Ask follow-up questions only when the athlete's answer is ambiguous or incomplete for a coaching note (e.g. "was that mental stop the same pattern as before, or different?").

### Step 3: Offer performance comparison (optional but recommended)
When relevant, offer to compare lap times to previous sessions of the **same discipline**. Always enforce the rule: **only compare DYN to DYN, DYNb to DYNb, etc.**

To extract lap times from `workout_state.states`:
- Keys follow pattern `"ex-N"[reps]: t1,t2,...,tN` where each value is seconds per pool length
- Use `pool_length_meters` to compute speed: `speed = pool_length / lap_time`
- Present as a table grouped by series, showing 25m split times

### Step 4: Show notes before saving (when requested)
If the athlete says "show me before saving" or similar, display the draft note and wait for approval.

### Step 5: Save the workout
Call `Freediver:updateWorkout` with:
- `completed: true`
- `notes`: the coach-meaningful summary (in the athlete's preferred language)
- `rpe`: as a **number** (not string) — this is critical, passing a string will cause an error

## Note Writing Guidelines

**Structure** (adapt to what's relevant — omit sections with nothing meaningful):
```
[Main effort / opening distance]: [distance or time], [limiter or surplus], [mental state if notable].

[Repeating sets]: [CO₂ pattern], [any hypoxia], [completed / shortened].

Technique: [specific observations — discoveries, regressions, cues to remember].

[Performance comparison if done]: [key insight from the data].
```

**Language:** Match the athlete's language (Polish, English, etc.).

**Tone:** Factual and coach-facing. Avoid adjectives that don't add information ("great session"). Include the athlete's own hypotheses when they offer one (label as "athlete's hypothesis").

**What makes a note valuable:**
- Mental limiters clearly described (sudden stop vs fatigue vs fear)
- New technique discoveries with specific body part cues
- Unusual physical sensations (e.g. diaphragm fatigue, heavy legs) with cause if known
- CO₂ / hypoxia onset timing (e.g. "CO₂ from 75m", "hypoxia signs on last 25m")
- Adaptation within a session (e.g. contractions moved from 2min to 3min across reps)
- Subjective form assessment ("felt like +200m surplus")

**What to omit:**
- "All reps completed" when that's the plan
- Causes that the athlete did NOT mention (don't invent context like "probably tired from gym")
- Comparisons across disciplines

## Parsing workout_state.states

The states object may be stored in a large file. Use bash to parse it:

```python
import json, re
with open('<path>') as f:
    text = json.load(f)[0]['text']

blocks = re.findall(r'\[(\d+)\]:\n(.*?)(?=\[\d+\]:\n|\Z)', text, re.DOTALL)
for num, b in blocks:
    name_m = re.search(r'name: ([^\n]+)', b)
    date_m = re.search(r'scheduled_at: \"([^\"]+)\"', b)
    states_m = re.search(r'states:\n((?:\s+\"ex-\d+\".+\n)+)', b)
    # filter by discipline keyword in block
```

States format: `"ex-N"[count]: t1,t2,...` — each value is seconds for one pool length.

## Common Pitfalls

- **RPE must be a number**: `rpe: 3` not `rpe: "3"` — passing a string returns an error
- **Don't invent context**: Only write what the athlete said in this conversation
- **Don't compare across disciplines**: DYN ≠ DYNb — always fetch same-type historical workouts
- **Large result files**: `getWorkouts` results may be stored to disk — always check for the file path and parse with bash/python
- **Timezone**: Always call `user-profile` first; use the IANA timezone for all date ranges
- **updateWorkout may fail with "Denied"**: If so, display the note to the athlete and ask them to copy it manually