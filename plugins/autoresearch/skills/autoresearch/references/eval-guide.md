# Eval Guide

How to write eval criteria that actually improve your skills instead of giving you false confidence.

---

## the golden rule

Every eval must be a yes/no question. Not a scale. Not a vibe check. Binary.

Why: Scales compound variability. If you have 4 evals scored 1-7, your total score has massive variance across runs. Binary evals give you a reliable signal.

---

## good evals vs bad evals

### Text/copy skills (newsletters, tweets, emails, landing pages)

**Bad evals:**
- "Is the writing good?" (too vague — what's "good"?)
- "Rate the engagement potential 1-10" (scale = unreliable)
- "Does it sound like a human?" (subjective, inconsistent scoring)

**Good evals:**
- "Does the output contain zero phrases from this banned list: [game-changer, here's the kicker, the best part, level up]?" (binary, specific)
- "Does the opening sentence reference a specific time, place, or sensory detail?" (binary, checkable)
- "Is the output between 150-400 words?" (binary, measurable)
- "Does it end with a specific CTA that tells the reader exactly what to do next?" (binary, structural)

### Visual/design skills (diagrams, images, slides)

**Bad evals:**
- "Does it look professional?" (subjective)
- "Rate the visual quality 1-5" (scale)
- "Is the layout good?" (vague)

**Good evals:**
- "Is all text in the image legible with no truncated or overlapping words?" (binary, specific)
- "Does the color palette use only soft/pastel tones with no neon, bright red, or high-saturation colors?" (binary, checkable)
- "Is the layout linear — flowing either left-to-right or top-to-bottom with no scattered elements?" (binary, structural)
- "Is the image free of numbered steps, ordinals, or sequential numbering?" (binary, specific)

### Code/technical skills (code generation, configs, scripts)

**Bad evals:**
- "Is the code clean?" (subjective)
- "Does it follow best practices?" (vague, which best practices?)

**Good evals:**
- "Does the code run without errors?" (binary, testable — actually execute it)
- "Does the output contain zero TODO or placeholder comments?" (binary, greppable)
- "Are all function and variable names descriptive (no single-letter names except loop counters)?" (binary, checkable)
- "Does the code include error handling for all external calls (API, file I/O, network)?" (binary, structural)

### Performance optimization (Lighthouse, latency, bundle size)

**Bad evals:**
- "Is the site fast?" (vague — fast compared to what?)
- "Rate the performance 1-10" (scale = unreliable)
- "Did performance improve?" (relative — need a fixed threshold)

**Good evals (command evaluators):**
- `command: "npx lighthouse ... --output=json"` / `extract: ".categories.performance.score"` / `check: ">= 0.9"` (Lighthouse performance above 90)
- `command: "npx lighthouse ... --output=json"` / `extract: ".audits.largest-contentful-paint.numericValue"` / `check: "< 2500"` (LCP under 2.5s)
- `command: "du -sb dist | cut -f1"` / `extract: "raw"` / `check: "< 204800"` (bundle under 200KB)
- `command: "hey -n 100 -c 10 http://localhost:8000/api/endpoint -o json"` / `extract: ".avg_latency_ms"` / `check: "< 100"` (avg response under 100ms)

**Good evals (judgment, for non-measurable qualities):**
- "Does the page still render all original visible content sections?" (guards against optimizing away actual content)
- "Are all interactive elements (buttons, forms, links) still functional?" (guards against breaking UX for speed)
- "Does the API response still contain all required fields from the original schema?" (guards against returning less data for speed)

**Important:** Always pair command evaluators (measuring performance) with at least one judgment evaluator (verifying nothing was broken). Optimizing speed is easy if you're allowed to delete features.

### Document skills (proposals, reports, decks)

**Bad evals:**
- "Is it comprehensive?" (compared to what?)
- "Does it address the client's needs?" (too open-ended)

**Good evals:**
- "Does the document contain all required sections: [list them]?" (binary, structural)
- "Is every claim backed by a specific number, date, or source?" (binary, checkable)
- "Is the document under [X] pages/words?" (binary, measurable)
- "Does the executive summary fit in one paragraph of 3 sentences or fewer?" (binary, countable)

---

## common mistakes

### 1. Too many evals
More than 6 evals and the skill starts gaming them — it optimizes for passing the test instead of producing good output. Like a student who memorizes answers without understanding the material.

**Fix:** Pick the 3-6 checks that matter most. If everything passes those, the output is probably good.

### 2. Too narrow/rigid
"Must contain exactly 3 bullet points" or "Must use the word 'because' at least twice" — these create skills that technically pass but produce weird, stilted output.

**Fix:** Evals should check for qualities you care about, not arbitrary structural constraints.

### 3. Overlapping evals
If eval 1 is "Is the text grammatically correct?" and eval 4 is "Are there any spelling errors?" — these overlap. A grammar fail often includes spelling. You're double-counting.

**Fix:** Each eval should test something distinct.

### 4. Unmeasurable by an agent
"Would a human find this engaging?" — an agent can't reliably answer this. It'll say "yes" almost every time.

**Fix:** Translate subjective qualities into observable signals. "Engaging" might mean: "Does the first sentence contain a specific claim, story, or question (not a generic statement)?"

---

## writing your evals: the 3-question test

Before finalizing an eval, ask:

1. **Could two different agents score the same output and agree?** If not, the eval is too subjective. Rewrite it.
2. **Could a skill game this eval without actually improving?** If yes, the eval is too narrow. Broaden it.
3. **Does this eval test something the user actually cares about?** If not, drop it. Every eval that doesn't matter dilutes the signal from evals that do.

---

## template

Copy this for each eval:

```
EVAL [N]: [Short name]
Question: [Yes/no question]
Pass: [What "yes" looks like — one sentence, specific]
Fail: [What triggers "no" — one sentence, specific]
```

Example:

```
EVAL 1: Text legibility
Question: Is all text in the output fully legible with no truncated, overlapping, or cut-off words?
Pass: Every word is complete and readable without squinting or guessing
Fail: Any word is partially hidden, overlapping another element, or cut off at the edge
```

---

## command evaluator format

Command evaluators use three fields:

```
command:  shell command that produces output
extract:  how to get the numeric value:
          - jq-style path for JSON: ".categories.performance.score"
          - "field N" for whitespace-delimited: "field 1"
          - "raw" for plain numeric output
check:    threshold comparison: ">= 0.9", "< 5242880", "< 100"
```

The agent runs the command, extracts the value, and compares against the threshold. Pass or fail — no scales.

When multiple evaluators share the same command string, the command runs once per run and all evaluators parse the same output.
