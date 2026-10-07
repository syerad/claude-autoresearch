# Eval Guide

How to write eval criteria that actually improve your skills instead of giving you false confidence.

---

## the golden rule

Every eval must be a yes/no question. Not a scale. Not a vibe check. Binary. The one exception is a **metric objective** — a raw number a command produces (latency, runtime, size, a Lighthouse score) — which is compared directly against the current best.

Why: Scales compound variability. If you have 4 evals scored 1-7, your total score has massive variance across runs. Binary evals give you a reliable signal. A number a machine measures is not a scale someone invents, and thresholding it throws information away.

---

## objectives and constraints

Each run has exactly one **objective** — what it improves — and any number of **constraints** — what must not break.

- **Metric objective** when the quality is a number. Compare the raw value against the current best, direction-aware, with the measured noise margin. No thresholds, so no sub-threshold blindness.
- **Pass-count objective** when the quality is a set of yes/no properties: the total passes of the scored checks.
- **Constraints** are yes/no checks that must pass in every run; one failure discards the candidate. Correctness belongs here: output identical to a golden file, content still present, no banned phrases. Every metric objective needs at least one — optimizing speed is easy if you're allowed to delete features.

A run with no roles assigned — every check scored, no constraints — is a pass-count objective over all of them.

---

## calibrating thresholds

If the quality is a number, make it a metric objective and this section doesn't apply. Otherwise: binary evals are blind to movement that doesn't cross the threshold. If the baseline Lighthouse score is 0.62 and your only check is `>= 0.9`, then a mutation that reaches 0.85 scores identically to one that did nothing — and the loop discards it. Repeat that a few times and the run stalls while real progress gets thrown away.

Two fixes:

- **Set thresholds just beyond the current baseline.** Measure first, then check for the next meaningful step (baseline 0.62 → check `>= 0.7`). Tighten the threshold in a follow-up run once it passes consistently.
- **Stepped thresholds.** Use the same command as multiple evaluators with increasing checks — `>= 0.7`, `>= 0.8`, `>= 0.9`. Each increment flips one more eval, so incremental improvement registers as score movement. Command deduplication means this costs nothing extra: the command still runs once per run.

The fixed-threshold rule below ("Did performance improve?" is a bad eval) still holds — the threshold must be absolute, not relative. Calibration is about *where* you place the absolute line, not about making it relative.

---

## who judges judgment evals

A judgment eval is only as reliable as the judge's independence. The agent that authored a mutation must never grade it — it knows what it changed, why, and wants it kept. Self-graded judgments say "yes" almost every time.

- **Judge blind.** Dispatch a fresh subagent that receives only the artifact (the rendered page, the generated diagram, the command output) and the yes/no question. Never include the diff, the hypothesis, or the changelog.
- **Ask a question the artifact alone can answer.** "Is the progress output still clear?" needs the old output the judge never sees; "Does every progress line show a percentage, a row count, and an ETA?" doesn't. If a judgment needs a reference — a section inventory, a required-fields list — write it into the artifacts directory at setup and give it to the judge alongside the artifact.
- **Get a reason with every verdict.** The judge answers `PASS` or `FAIL` and one sentence why. The reasons are the best feedback the agent making changes gets about *why* something fails — give it the dev-task reasons, never the held-out ones. Blindness is about what the judge sees, not about what happens to its answer.
- **Ground the judgment in a fresh artifact.** Every run must produce the thing being judged — execute the skill against a fixed test-prompt set, fetch the page, run the binary. A judgment with nothing fresh to inspect measures optimism, not quality.
- **Prefer command evals when the check is mechanical.** "Is the output identical to the saved baseline?" is a `diff | wc -l` command eval, not a judgment. Reserve judgments for qualities a script can't check.

---

## calibrating judges before the run

A judge that disagrees with the user will optimize the target toward something they don't want, and nobody finds out until the end. Before setup, calibrate every judgment eval on the user's labelled examples:

1. Write down the verdict each example should get on each eval — good examples pass every eval; a bad example fails the evals it was collected to illustrate — and have the user correct the table.
2. Run 3 blind judges per eval per example.
3. The eval passes if all 3 verdicts match on every example. Otherwise rewrite the question (show the user the judges' reasons — they usually point at the ambiguity) and calibrate again. After two failed rewrites, drop the eval or hand it to the user.

Then freeze the wording and the judge model. Changing either mid-run means re-running the baseline: scores from before and after are measured with different rulers.

---

## measuring noise

Don't assume how noisy an eval is — measure it. At setup, evaluate the unchanged baseline three times over:

- **A check that flips** on an unchanged target is noisy. As a constraint it would discard good changes at random, so it can't be one: make it deterministic, or move it into the score.
- **A scored check that always passes** carries no signal. Drop or tighten it.
- **The spread** of the three baseline totals (pass-count) or means (metric) becomes the noise margin: the minimum improvement that counts. Pass-count runs with any judgment keep a floor of 2.

Measuring once is still not enough for a keep: the candidate that scored highest was partly lucky by selection. Every would-be keep is re-measured alongside the anchor, fresh, and the fresh numbers decide.

---

## held-out tasks

When the target is evaluated over a set of inputs — test prompts, benchmark files — set aside about a third, at least 2, that the agent making changes never sees: not the inputs, not the outputs, not the judges' reasons. Instructions alone don't achieve that — whatever a subagent returns lands in the dispatching agent's context — so the separation is structural: a subagent creates the held-out set, subagents run the target on it and write outputs to files, judges read those files and write bare verdicts to files, and a tally subagent compares against the baseline and returns only `held` or `regressed` — the only held-out word that reaches the agent or the logs it re-reads. Score them at the baseline, on every candidate that survives re-measurement, and at the end. A candidate that gains on the visible tasks while doing worse on the held-out ones is memorizing the exam, and is discarded.

---

## keeping evals out of reach

An eval only measures the target while nothing else changes. Agents under pressure to raise a number will, sooner or later, edit the test, the golden file, or the benchmark instead of the code. So:

- Target files and eval inputs never overlap. If a test file is both, the run can't measure it — split it or drop it from the evals.
- After every mutation commit, the worktree's `git status --porcelain` must match what it was before the edit; anything else touched means the mutation is discarded. After the guards and after evaluation, no tracked file may have changed either — a guard or evaluator that rewrites files would put an unmeasured change under the candidate.
- Guard and evaluator commands live in checksummed scripts under `commands/`, so a mutation can't change what measures it.
- Eval inputs outside the worktree (`tasks/`, `holdout/`, `examples/`) are checksummed at setup and verified before every evaluation.

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
- `command: "du -sk dist | cut -f1"` / `extract: "raw"` / `check: "< 200"` (bundle under 200KB — `du -sk` is portable, GNU-only `du -sb` is not)
- `command: "hey -n 100 -c 10 http://localhost:8000/api/endpoint | awk '/Average:/{print $2*1000}'"` / `extract: "raw"` / `check: "< 100"` (avg response under 100ms — hey has no JSON output; parse its text summary)

**Good evals (judgment, for non-measurable qualities):**
- "Does the page render every section listed in `tasks/sections.md`?" (guards against optimizing away actual content — the inventory is written at setup, so the judge needn't know the original)
- "Do the buttons, forms, and links listed in `tasks/interactions.md` all respond?" (guards against breaking UX for speed)
- "Does the API response contain every field listed in `tasks/schema.json`, each with the listed type?" (guards against returning less data for speed)

**Important:** Always pair a performance objective with at least one constraint verifying nothing was broken — a golden-output `diff` where the output is deterministic, a judgment where it isn't. Optimizing speed is easy if you're allowed to delete features.

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

### 5. Gameable evals
An eval the target can pass without getting better will be passed that way, given enough experiments — by special-casing the test input, detecting the benchmark, or deleting the feature the eval doesn't look at.

**Fix:** before the run, a refuter attacks the eval set and proposes edits that fool it; block each plausible one with a constraint. See [refutation.md](refutation.md).

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
command:   shell command that produces output
extract:   how to get the numeric value:
           - jq-style path for JSON: ".categories.performance.score"
           - "field N" for whitespace-delimited: "field 1"
           - "raw" for plain numeric output
check:     threshold comparison: ">= 0.9", "< 5242880", "< 100"
direction: lower | higher — a metric objective only, instead of check
```

The agent runs the command, extracts the value, and compares against the threshold. Pass or fail — no scales. For a metric objective, the extracted value itself is the result.

When multiple evaluators share the same command string, the command runs once per run and all evaluators parse the same output.
