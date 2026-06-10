---
name: autoresearch
description: "Autonomously optimize any target — skill prompts, application code, copy, or documents — by running an iterative mutation/evaluation loop. Define target files, evaluators (shell commands with thresholds and/or binary agent judgments), and guards. The agent mutates, measures, keeps or discards, and repeats. Based on Karpathy's autoresearch methodology. Use when: optimize this skill, improve this skill, run autoresearch on, make this skill better, self-improve skill, benchmark skill, eval my skill, run evals on, optimize this code, improve performance, optimize for lighthouse, reduce bundle size, speed up this endpoint, autoresearch this codebase, improve this copy, optimize this prompt, give me variants of, explore scenarios for."
---

# Autoresearch

Autonomously optimize any target — skill prompts, frontend performance, API latency, bundle size, marketing copy, forecasts, or anything else you can measure. Define what to change, how to score it, and what must not break. The agent handles the rest.

This skill adapts Andrej Karpathy's autoresearch methodology to any optimization target. The core loop: mutate target files, run guards, evaluate against binary criteria, keep improvements, discard the rest.

---

## the core job

Take any set of target files, define what "good" looks like as binary pass/fail checks, then run an autonomous loop that:

1. Mutates the target files (one change per experiment)
2. Runs guard commands to verify nothing is broken
3. Scores the result against all evaluators
4. Keeps mutations that improve the score, discards the rest
5. Repeats until the score ceiling is hit, max iterations reached, or the user stops it

**Output:** Optimized target files (on a dedicated `autoresearch/[name]` branch for git runs) + `results.tsv` log + `changelog.md` of every mutation attempted + a live HTML dashboard. Depending on output mode, the deliverable is one winner, a shortlist of finalists, or a portfolio of distinct variants.

---

## before starting: the front-door

**STOP. Do not run any experiments until the setup is confirmed. This section describes how to reach that confirmed state.**

The skill opens with a single prompt to the user: *"What are you trying to improve?"*

From that one message, run four passes before presenting the setup to the user.

### Pass 1 — Classify (silent)

Extract signals from the user's message and any target files they mentioned:

- **Target type:** code / text / prompt / binary / live system
- **Target files:** exact paths if mentioned; otherwise inferred from context
- **Intent verb:** improve, optimize, explore, compare, pick, find the best, give me options
- **Quality dimensions:** speed, tone, accuracy, cost, length, defensibility, etc.
- **Explicit numbers/thresholds:** anything the user stated as a target ("< 100ms", "under 200 words")

If target files were mentioned or inferred, **read them** before continuing. Skim for context: architecture for code, tone and structure for text, assumption cells for a forecast.

Classify three axes:

1. **Target type** → picks the half of [references/eval-patterns.md](references/eval-patterns.md) to pull from and the mode default.
2. **Artifact type** (text / binary / live / prompt-that-generates) → picks the rollback mechanism per [references/rollback-mechanisms.md](references/rollback-mechanisms.md).
3. **Intent verb** → picks the output mode default per [references/output-modes.md](references/output-modes.md).

### Pass 2 — Explain

In one paragraph of plain language, tell the user what autoresearch will do with their target. First-mention glossing applies — every concept introduced here gets a ≤15-word explanation:

- *"Evals are pass/fail checks that score each experiment. You usually want 3–6."*
- *"Guards are quick sanity checks that must pass; if they don't, the change is thrown out."*
- *"Rollback means: if a change makes things worse, we automatically undo it."*

Example paragraph for a cold-email copy target:

> *"I'll run small changes to `templates/cold-email.md` one at a time, check each against 3–5 quality rules you pick, and keep only the versions that improve the score. I work on a separate git branch, and anything that makes things worse is automatically undone — the rest of your repo isn't touched. You can stop anytime."*

### Pass 3 — Present the draft

Based on Pass 1, produce a full draft configuration. Every field is marked ✓ (confident) or ? (needs user input). Present it in a compact block:

```
Run name:            cold-email                                        ✓
Target files:        templates/cold-email.md                           ✓
Target type:         text / writing                                    ✓
Rollback:            git (branch autoresearch/cold-email)              ✓
Output mode:         top-3 — you asked for "options"                   ✓

Evals (drafted — review, edit, remove, or add):
  1. Opening specificity (judgment)                                    ?
     — Your good examples all open with a specific time/place. Check.
  2. Single concrete ask at the end (judgment)                         ?
     — Your good examples end with a specific ask. Check.
  3. Length 40–80 words (command: wc -w)                               ?
     — Inferred from your good examples.
  4. Banned phrases: "game-changer", "level up", "touch base"          ?
     — Extracted from your bad examples.

Guards:              markdown lint (markdownlint)                      ✓
Timeout:             120s per experiment (guards + all runs)           ✓
Max iterations:      20                                                ✓
Runs per experiment: 5 (non-deterministic output)                      ✓
```

For each proposed eval, include a **Grounded in:** line — either the user's message, their examples, or "(generic — edit to match your taste)". This is non-optional.

### Pass 4 — Gap-fill

For every `?` item, ask exactly one question. Prefer multiple choice. Each question includes a "why we're asking" line.

Example:

> *"I need 2–3 examples of intros you'd send and 1–2 you'd delete. Why: your examples let me write evals that match your taste, not a generic 'good writing' template. Paste them below, or say 'none' and I'll propose generic evals you can edit."*

Keep asking until every `?` is resolved. When the draft is fully ✓, move to the setup checklist below.

### Inline education rules

- First mention of a concept = ≤15-word gloss. Second mention = no gloss. Tracked per session.
- Concepts to gloss: baseline, eval, guard, mutation, rollback, score, pass rate, iteration, keep, discard, variant, diversity dimension.
- Every proposed eval has a `Grounded in:` line.
- At every step, offer two escape hatches: *"skip this, use a default"* and *"I want to edit directly"* (drops the user into the raw configuration form below).

### the configuration fields (produced by the front-door)

Pick a short kebab-case run name `[name]` (e.g. `nextjs-perf`, `cold-email`) — it names the branch, the rollback anchor, and the artifacts directory. The front-door must populate all of the following. Power users can say "I want to edit directly" and fill them in raw.

1. **Target files** — Explicit list of file paths the agent can edit. Nothing else is editable.

2. **Evaluators** — List of checks (3-6 recommended), drawn from [references/eval-patterns.md](references/eval-patterns.md) and grounded in user examples where possible (see [references/eval-guide.md](references/eval-guide.md) for principles). Each one is either:

   **Command evaluator** — a shell command, extraction path, and threshold:
   ```
   command:  "npx lighthouse http://localhost:3000 --output=json --quiet"
   extract:  ".categories.performance.score"
   check:    ">= 0.9"
   ```
   Extraction formats: jq-style path for JSON (`.field.subfield`), `"field N"` for whitespace-delimited output, `"raw"` for plain numeric output.

   **Calibrate thresholds against the current baseline.** Binary scoring is blind to sub-threshold movement: if the baseline is 0.62 and the threshold is `>= 0.9`, an improvement to 0.85 scores exactly the same as no improvement and gets discarded. Either set thresholds just beyond the current value so progress can register, or use stepped thresholds — the same command as three evaluators with `>= 0.7`, `>= 0.8`, `>= 0.9` — so each increment flips an eval.

   **Judgment evaluator** — a binary yes/no question. Two things must be pinned down at setup:
   - **What is judged and how it is produced fresh each run.** For code: fetch the page, run the binary, read the build artifact. For skill/prompt targets: write a fixed set of 3-5 test prompts into `autoresearch-[name]/tasks/` at setup; each run executes the target against them in a fresh subagent. A judgment with no fresh artifact to inspect is ungrounded — the score would just measure the agent's optimism about its own edit.
   - **Who judges: a fresh subagent, blind to the experiment.** Dispatch a subagent that receives ONLY the artifact and the yes/no question — never the diff, the hypothesis, or the changelog. The agent that authored a mutation must not grade it; self-graded judgments say "yes" almost every time.

   Both types can be mixed in a single run. Prefer command evaluators wherever the quality is mechanically checkable (a deterministic "is the output identical" check belongs in a `diff`-based command eval, not a judgment).

   **Command deduplication:** When multiple evaluators share the same command string, the command runs once per run. All evaluators sharing that command parse the same output.

3. **Guards** — Shell commands that must exit 0 after every mutation (at least one required). If any guard fails, the mutation is auto-discarded without running evaluators.
   - Code: `npm run build`, `npm test`, `pytest tests/`
   - Non-code targets without a natural build step: `echo ok`, or a structural check like `markdownlint`

4. **Timeout** — Max seconds per experiment (required). The budget covers one full experiment: all guard commands plus all N evaluation runs combined. If exceeded, the experiment is auto-discarded. See "enforcing the timeout" below. Rule of thumb: `guard time + runs × (sum of unique evaluator command times) + 20% margin`.
   - Lighthouse, 3 runs: ~300s
   - API benchmarking, 3 runs: ~240s
   - Docker rebuild + integration tests: ~600s
   - Skill/prompt optimization, 5 LLM runs: ~600s

5. **Max iterations** — Max experiment cycles before stopping (required). Forces you to choose a compute budget. Can be set high (100) but must be explicit.

6. **Runs per experiment** — How many times to evaluate per mutation. Defaults to 5 if unspecified. 3 is fine for deterministic benchmarks. 5 for nondeterministic outputs (skill prompts, LLM judgments).

Plus two front-door outputs:

7. **Output mode** — single-winner / top-N / exploration, per [references/output-modes.md](references/output-modes.md). Exploration additionally needs 1–3 diversity dimensions with thresholds (no dimension nameable → run top-N instead).

8. **Rollback mechanism** — git / snapshot-dir / API / manual-confirm, per [references/rollback-mechanisms.md](references/rollback-mechanisms.md). One mechanism per run, covering every target.

### scoring

Everything is binary — pass or fail. Command evaluators extract a value and check it against the threshold. Judgment evaluators are yes/no.

**Total score** = passes across all evaluators × all runs.
**Max score** = number of evaluators × runs per experiment.
**Pass rate** = score / max_score.

Each "run" executes all unique commands once (deduplicated), then scores all evaluators against the output. So 4 evaluators using 2 unique commands with 3 runs = 6 command executions total, 12 pass/fail scores. Max score = 12.

**Failed commands:** if a command exits non-zero, is killed by the timeout, or its extraction yields no numeric value, every evaluator bound to that command scores fail for that run. Never substitute a stale or guessed value.

**Experiments rejected before scoring** (guard failure or timeout) are never evaluated: log them with empty score/max_score/pass_rate fields in `results.tsv` and `null` pass_rate in the dashboard data. Do not fabricate a score for an experiment that was never measured.

In **exploration mode**, evals are hard constraints — a candidate must pass all of them to be kept, and score is replaced by a distinctness check against existing kept variants. See [references/output-modes.md](references/output-modes.md).

---

## setup

Once the front-door has confirmed all fields, run these steps in order. The git mechanism is shown inline (the common case); for snapshot-dir, API, and manual-confirm the same steps dispatch to the per-mechanism operations in [references/rollback-mechanisms.md](references/rollback-mechanisms.md).

1. **Confirm configuration** — all fields populated from the front-door: target files, evaluators, guards, timeout, max iterations, runs per experiment, output mode, rollback mechanism, run name.
2. **Safety-review guards and evaluators.** These commands run unattended, dozens of times. Refuse or explicitly confirm anything destructive or outward-facing: deletes outside the artifacts dir, `sudo`, piping downloads to a shell, benchmarks pointed at production URLs.
3. **Run the rollback pre-flight** per [references/rollback-mechanisms.md](references/rollback-mechanisms.md). For git that means: resolve the target repo (`git -C <dir-of-target> rev-parse --show-toplevel` — ALL targets must resolve to the SAME repo, and every git command below runs as `git -C <target repo>`); clean working tree (abort on uncommitted changes — do NOT auto-commit user's work); not detached HEAD; no collision with a previous run's tag, branch, or artifacts dir (offer resume or a new name — never overwrite a previous run's `baselines/`). If any pre-flight fails, abort with that doc's message.
4. **Read and understand all target files.** For code: architecture, dependencies, what each file does. For writing: tone, structure, intended audience. For forecasts: assumption cells, formula chains, which cells feed the outputs the user cares about.
5. **Verify guards pass** on the current state, each prefixed with `timeout`. If any fails, stop and tell the user — the target is already broken.
6. **Create the run branch** (git): `git -C <target repo> checkout -b autoresearch/[name]`. Every experiment commit lands here; the user's branch is never touched.
7. **Create the working directory** `autoresearch-[name]/` at the target repo root. Add `autoresearch-*/` to `.gitignore` if not already present — and if that changed `.gitignore`, commit that single change now (`autoresearch: ignore artifacts directory`). The ignore entry must be committed before the first experiment, or a later discard's reset will unignore the artifacts and a subsequent commit-and-discard cycle will delete them from disk.
8. **Back up all target files** to `autoresearch-[name]/baselines/`, mirroring their repo-relative paths. These are the pre-run originals — an abort escape hatch, not the rollback mechanism, and identical across all mechanisms.
9. **Establish the rollback anchor.** Git: `git -C <target repo> tag -f autoresearch/[name]/good`. Snapshot-dir: copy targets to `iterations/0000-baseline/` and `0000-good/`. API: export and record iteration 0. Manual-confirm: log the baseline and get the user's ack.
10. **Create artifacts:** `results.tsv` (with header row), `changelog.md` (empty), `dashboard.html` (copy from [references/dashboard-template.html](references/dashboard-template.html), replace `__DATA_PLACEHOLDER__` with initial data JSON including the `mode` field), for top-N runs `scores.json`, and for skill/prompt targets the fixed test-task set under `autoresearch-[name]/tasks/`. Open the dashboard: `open autoresearch-[name]/dashboard.html` (macOS) / `xdg-open` (Linux).
11. **Run baseline** (experiment 0) — evaluate the current state without changing anything. Score all evaluators × all runs. Then:
    - 100% → inform user all evals already pass, stop.
    - max_score − 1 → warn the user the run is one pass from the ceiling and ask whether it's worth proceeding.
    - Otherwise → report the baseline score and proceed. Do not wait for an acknowledgment — the user may already be away.

    In **exploration mode**, the baseline must pass ALL evals — it becomes variant `base` and counts toward N. If it fails any eval, abort: exploration requires a valid starting point.

---

## the experiment loop

Once the baseline is reported, the loop runs autonomously. Do not pause to ask the user between experiments. (Exception: the manual-confirm rollback mechanism pauses for the user's snapshot ack each iteration — there, the pause IS the mechanism.)

**LOOP:**

**0. Re-read** — Read `results.tsv` and the last 10 entries of `changelog.md`. Skip on first iteration. This keeps experimental context alive across context window compression.

**1. Analyze** — Which evals fail most? Read the actual outputs or command results that failed. Identify the pattern: is it a formatting issue? A performance bottleneck? A missing optimization? An ambiguous instruction?

**2. Hypothesize** — Pick ONE thing to change. Do not change 5 things at once — you won't know what helped.

Good mutations:
- Add a specific code change or instruction addressing the most common failure
- Refactor unclear code / reword an ambiguous instruction
- Add a guard clause or anti-pattern for a recurring mistake
- Reorder code/instructions for priority (position matters in prompts)
- Add or improve an example showing correct behavior
- Remove something causing over-optimization for one eval at the expense of others
- Simplify — a score-preserving simplification is kept under the tiebreak in step 7
- Exploration mode: target a different region of the solution space along an unexplored diversity-dimension value

Bad mutations:
- Rewriting everything from scratch
- Changing 5 things at once
- Adding complexity without a specific hypothesis
- Vague changes ("make it better")

**3. Verify the anchor** — Confirm the rollback anchor matches the current target state (git: `autoresearch/[name]/good` points at HEAD). It always should: KEEP advances it and DISCARD resets to it. If they diverge, something went wrong — stop and run recovery instead of force-tagging over the discrepancy. (Exploration: the anchor is the base state — every candidate starts from it.)

**4. Mutate** — Edit target file(s), then record the attempt per the rollback mechanism. Git: stage ONLY the declared target files (`git -C <target repo> add <each target path>` — never `git add -A`; sweeping in artifacts or unrelated files is how a later discard deletes them) and commit as a single commit with `--no-verify` (a pre-commit hook that rewrites files would silently change the mutation; one that rejects the message format would derail the loop). Message format: `autoresearch: [short description of change]`. Verify the target files are clean in `git status` after committing. Snapshot-dir: copy the mutated targets to `iterations/NNNN-attempt/`.

**5. Guard** — Run all guard commands, each prefixed with `timeout <remaining>s` (see "enforcing the timeout").
- Any guard fails → roll back to the anchor (git: `git -C <target repo> reset --hard autoresearch/[name]/good`), log as `guard_fail`, go to step 0.
- Budget exceeded or a command killed (exit 124) → roll back the same way, log as `timeout`, go to step 0.

**6. Evaluate** — Run all evaluators × N runs, every command timeout-prefixed, tracking the remaining experiment budget. For each run: execute all unique commands once (deduplicated), then score every evaluator against the output. For judgment evaluators: produce the fresh artifact, then dispatch a blind subagent with only the artifact and the question. If the budget runs out mid-evaluation, treat the experiment as `timeout` (roll back, log, go to step 0).

**7. Decide** — by output mode:

- **single-winner:**
  - Score improved — and, if any evaluator is a judgment or otherwise non-deterministic, improved by **at least 2 passes** → **KEEP.** A +1 blip on a noisy eval is indistinguishable from variance; treating it as a win ratchets noise into the baseline. (With only deterministic command evals, any strict improvement counts.)
  - Score equal, the change strictly simplifies the target files (net-negative diff), and all guards passed → **KEEP** as a simplification win. Note it as such in the description.
  - Otherwise → **DISCARD.** Roll back to the anchor.
  - **On KEEP, immediately advance the anchor** (git: `git -C <target repo> tag -f autoresearch/[name]/good`). The anchor must always point at the current baseline — if the run stops right after a keep, a stale anchor makes recovery silently destroy the best result.

- **top-N:** identical to single-winner during the loop. Additionally, append every evaluated experiment's per-eval pass counts to `scores.json` — finalist selection needs them. Selection happens after the loop stops, per [references/output-modes.md](references/output-modes.md).

- **exploration:**
  - All guards pass AND all evals pass AND the candidate differs from every existing kept variant on at least one diversity dimension → **KEEP as a new variant**: record its anchor (git: tag `autoresearch/[name]/variant-K`), then return the targets to the base state (git: `reset --hard autoresearch/[name]/good`) so the next candidate also mutates from base.
  - Any eval fails OR not distinct → **DISCARD.** Roll back to the base anchor.
  - "Score improvement" is not the decision rule — validity plus distinctness is.

**8. Log** — Append to `results.tsv` (exploration keeps use a `variant K — …` description). Update `dashboard.html` with new inline data. Append to `changelog.md` (see format below). Top-N: also `scores.json`.

**9. Check stop conditions:**

- User manually stops → stop, deliver results.
- Max iterations reached → stop, deliver results.
- **single-winner / top-N — ceiling reached:** the standing baseline score is ≥ max_score − 1 AND the last 3 consecutive experiments (kept or discarded) failed to improve it → stop, deliver results. Judge this against the baseline's own score — never against the logged score of a discarded mutation, which is lower by definition.
- **exploration:** N distinct valid variants found (counting `base`) → stop, deliver results.
- (Per-experiment timeout is handled in steps 5-6 — that experiment is discarded, loop continues.)
- Otherwise → go to step 0.

**NEVER STOP between experiments to ask the user.** They may be away. Run autonomously until a stop condition is met or the user interrupts.

**If you run out of ideas:** Re-read the failing outputs. Re-read the changelog. Try combining two previous near-miss mutations. Try a completely different approach to the same problem. Try removing things instead of adding them — a simplification that holds the score is a keepable win.

### enforcing the timeout

An agent cannot interrupt a hung shell command, so the timeout only exists if every command is wrapped. Record the experiment start time. Before each guard or evaluator command, compute the remaining budget and prefix the command with `timeout <remaining>s` (GNU coreutils; on macOS `brew install coreutils` provides it as `timeout` or `gtimeout`). Exit code 124 means the command was killed — treat it as the budget being exceeded. Never run a guard or evaluator bare: a mutation can pass every guard and still hang at runtime (an accidental infinite loop passes `py_compile` and unit tests that don't call the hot path), and an unwrapped evaluator would block the loop forever.

---

## rollback strategy

Full per-mechanism details in [references/rollback-mechanisms.md](references/rollback-mechanisms.md). Principles that hold for every mechanism:

- **One change per experiment.** All target-file edits land in a single commit (git) / a single snapshot dir (snapshot-dir) / a single export (API) / a single user-confirmed change (manual-confirm).
- **The anchor lives forward.** After every KEEP, advance the anchor — the `autoresearch/[name]/good` tag, the latest `-good` dir, the most recent kept `export_id`. Never depend on `HEAD~1` or similar relative references.
- **Rollback is atomic per experiment.** A discard returns ALL target files to the anchor state in one operation — a mechanism that can only revert some of the targets is the wrong mechanism for the run.
- **Clean state required at setup.** Pre-flight checks must pass before the loop starts.

For git specifically: dedicated run branch (`autoresearch/[name]`) so the user's branch never sees a mutation commit; per-run namespaced tag; stage only the declared target files; every git command runs against the resolved target repo (`git -C <target repo> …`), which is not necessarily the cwd's repo. Commit/log message format for git and any log-based mechanism: `autoresearch: [short description]`.

### recovery

If the session ends mid-experiment (crash, disconnect, context limit):

1. Read `autoresearch-[name]/` to determine the rollback mechanism (presence of `api-state.json`, `manual-snapshots.md`, or `iterations/`; otherwise git).
2. Check whether target files differ from the anchor (git: `git -C <target repo> diff`). If dirty, roll back to the anchor (git: `reset --hard autoresearch/[name]/good`). Because the anchor advances on every KEEP, this restores the latest kept state and never loses kept work.
3. `baselines/` holds the PRE-RUN originals. Restoring from baselines is a full abort that discards every kept improvement — only do it if the user explicitly wants to abandon the run, and say so when offering it.
4. If abandoning, set the dashboard `status` to `"error"` so the auto-refreshing page stops claiming the run is live.
5. To resume: re-read `results.tsv` and `changelog.md`, then re-enter the loop at step 0.

---

## changelog format

After every experiment (kept or discarded), append to `changelog.md`:

```markdown
## Experiment [N] — [baseline/keep/discard/guard_fail/timeout]

**Score:** [X]/[max] ([percent]%)  (or "not scored" for guard_fail/timeout)
**Change:** [One sentence describing what was changed]
**Reasoning:** [Why this change was expected to help]
**Result:** [What actually happened — which evals improved/declined]
**Failing outputs:** [Brief description of what still fails, if anything]
```

This changelog is the most valuable artifact. It's a research log that persists WHY things worked or failed. The agent re-reads the last 10 entries at the start of each loop iteration. Keep descriptions free of tabs and newlines — they also go into `results.tsv`.

---

## artifacts

All artifacts live in `autoresearch-[name]/` at the target repo root (or the targets' common directory for non-git runs):

```
autoresearch-[name]/
├── dashboard.html          # live browser dashboard (auto-refreshes, data inlined)
├── results.tsv             # score log for every experiment
├── changelog.md            # mutation log with reasoning (WHY things worked/failed)
├── scores.json             # per-eval pass counts per experiment (top-N runs)
├── tasks/                  # fixed test prompts for judgment evals (skill/prompt targets)
├── iterations/             # snapshot-dir mechanism only (see rollback-mechanisms.md)
└── baselines/              # original target files before any changes (abort escape hatch)
    └── [mirrored paths]    # repo-relative paths, mirrored
```

### dashboard

Copy from [references/dashboard-template.html](references/dashboard-template.html). Replace `__DATA_PLACEHOLDER__` with the JSON data object. Rewrite the entire `dashboard.html` file with updated inline data after each experiment (avoids `file://` CORS issues with `fetch()`). The template includes `<meta http-equiv="refresh" content="10">` for auto-refresh.

Dashboard data structure (embedded as `<script>const DATA = {...}</script>` in the HTML — annotations here are documentation, not part of the JSON):

```json
{
  "name": "autoresearch-nextjs-perf",
  "status": "running",            // "running" | "complete" | "error"
  "mode": "single",               // "single" | "top-n" | "exploration" (absent = "single")
  "current_experiment": 7,
  "max_iterations": 30,
  "baseline_score": 33.0,         // pass-rate PERCENT (0-100) of experiment 0 — not a raw count
  "best_score": 83.0,             // highest pass-rate PERCENT among baseline + kept experiments
  "variants_target": 3,           // exploration only: the requested N
  "variants": [                   // exploration only: gallery data
    {"id": "base", "label": "base", "dimensions": {"growth_rate": "8%"}, "evals_passed": 3, "evals_total": 3}
  ],
  "experiments": [
    {
      "id": 0,
      "score": 4,                  // raw pass count
      "max_score": 12,
      "pass_rate": 33.0,           // percent; null for guard_fail/timeout rows
      "status": "baseline",
      "description": "original code — no changes"
    }
  ],
  "eval_breakdown": [
    {"name": "Lighthouse >= 90", "pass_count": 0, "total": 3},
    {"name": "LCP < 2.5s", "pass_count": 1, "total": 3}
  ]
}
```

`eval_breakdown` reflects the most recent full evaluation of the current baseline — not a cumulative tally across all experiments. The experiments table renders in every mode. When the loop stops, set `status` to `"complete"` (or `"error"` if the run was abandoned) so the dashboard shows a finished state.

### results.tsv

Tab-separated with columns: experiment, score, max_score, pass_rate, status, description.

Status values: `baseline`, `keep`, `discard`, `guard_fail`, `timeout`. Guard failures and timeouts were never scored — leave their score, max_score, and pass_rate fields empty. Exploration keeps prefix the description with `variant K —`.

```
experiment	score	max_score	pass_rate	status	description
0	4	12	33.0%	baseline	original code — no changes
1	6	12	50.0%	keep	dynamic import for Hero component
2	6	12	50.0%	discard	added priority to hero image — no change
3				guard_fail	aggressive tree shaking — build broke
4				timeout	full image optimization pipeline — exceeded budget
5	10	12	83.3%	keep	CSS modules + font optimization
```

---

## deliver results

When the loop stops (max iterations, ceiling hit, variants found, or user interrupts), present:

1. **Score summary:** Baseline score → Final score (% improvement). Exploration: variants found of N requested.
2. **Total experiments run:** kept / discarded / guard failures / timeouts
3. **Top 3 most impactful changes** (from the changelog)
4. **Remaining failure patterns** (what still fails, if anything)
5. **Mode-specific deliverable:** top-N — the finalist table (per-eval score vectors from `scores.json`) and the "which one?" prompt; exploration — the variant gallery with per-variant "what's different" summaries. See [references/output-modes.md](references/output-modes.md) for selection and materialization mechanics.
6. **Location of all artifacts**
7. **Integration (git):** the optimized state lives on the `autoresearch/[name]` branch. After any finalist/variant selection is materialized (the variant tags are what materialization restores from — don't delete them before the user has picked), delete the run tags (`git -C <target repo> tag -d autoresearch/[name]/good` and any `variant-K` tags), then offer the user: squash-merge into their original branch, push and open a PR, or leave the branch for review. Never merge without asking.

---

## examples

### example 1: optimizing a Next.js Lighthouse score

**Configuration:**
- Target files: `src/app/page.tsx`, `src/components/Hero.tsx`, `next.config.js`
- Output mode: single-winner / Rollback: git
- Guards: `npm run build`, `npm test`
- Evaluators:
  - command: `npx lighthouse http://localhost:3000 --output=json --quiet` / extract: `.categories.performance.score` / check: `>= 0.9`
  - command: `npx lighthouse http://localhost:3000 --output=json --quiet` / extract: `.audits.largest-contentful-paint.numericValue` / check: `< 2500`
  - command: `du -sk .next | cut -f1` / extract: `raw` / check: `< 5120` (kilobytes — `du -sk` is portable; GNU-only `du -sb` is not)
  - judgment: "Does the page still display all original content sections and interactive elements?" (a blind subagent fetches the page each run and answers)
- Timeout: 300s (build ~60s + 3 Lighthouse runs at ~60s each)
- Max iterations: 20
- Runs: 3

Note: The two Lighthouse evaluators share the same command string. Lighthouse runs once per run (3 total), not twice per run. Both evaluators parse the same output.

**Result:** Baseline 33% → Final 100% in 4 kept experiments. Key wins: dynamic imports for below-the-fold components, CSS modules replacing global styles, optimizeCss in next.config.js.

### example 2: optimizing a Python API response time

**Configuration:**
- Target files: `src/api/routes/search.py`, `src/api/db/queries.py`, `src/api/cache.py`
- Output mode: single-winner / Rollback: git
- Guards: `pytest tests/`, `python -c "from src.api import app"`
- Evaluators:
  - command: `hey -n 200 -c 10 'http://localhost:8000/api/search?q=test' | awk '/Average:/{print $2*1000}'` / extract: `raw` / check: `< 100` (avg latency, ms)
  - command: `hey -n 200 -c 10 'http://localhost:8000/api/search?q=test' | awk '/99% in/{print $3*1000}'` / extract: `raw` / check: `< 500` (p99 latency, ms)
  - command: `python -c "import tracemalloc; tracemalloc.start(); from src.api import app; print(tracemalloc.get_traced_memory()[1])"` / extract: `raw` / check: `< 52428800`
  - judgment: "Does the search endpoint still return correct, complete results for a variety of query terms?"
- Timeout: 240s
- Max iterations: 30
- Runs: 3

Note: `hey` has no JSON output mode — parse its text summary (`Average:` line, `99% in` distribution line) with awk. The two hey commands differ, so they are NOT deduplicated; to share one execution per run, `tee` the output to a temp file in the first command and parse the file in the second.

**Result:** Baseline 33% → Final 100% in 6 experiments. Key wins: Redis caching layer, explicit column SELECTs instead of SELECT *, PostgreSQL full-text search replacing LIKE queries, async parallelization of independent DB calls.

### example 3: optimizing a skill prompt

**Configuration:**
- Target files: `~/.claude/skills/diagram-generator/SKILL.md`
- Output mode: single-winner / Rollback: `~/.claude/skills` is usually not a git repo — offer `git init` on it, else snapshot-dir (see [references/rollback-mechanisms.md](references/rollback-mechanisms.md))
- Guards: `echo ok`
- Test set: 5 fixed diagram prompts written to `autoresearch-[name]/tasks/` at setup; each run executes the skill against all 5 in a fresh subagent
- Evaluators (each judged per generated diagram by a blind subagent that sees only the diagram and the question):
  - judgment: "Is all text legible with no truncated or overlapping words?"
  - judgment: "Uses only pastel/soft colors — no neon, bright red, or high-saturation?"
  - judgment: "Linear layout — left-to-right or top-to-bottom with no scattered elements?"
  - judgment: "Free of numbered steps, ordinals, or sequential numbering?"
- Timeout: 600s (5 runs × LLM generation time)
- Max iterations: 15
- Runs: 5

**Result:** Baseline 80% (16/20) → Final 95% (19/20) in 5 experiments. Key wins: specific hex codes replacing vague "pastel colors" instruction, explicit anti-numbering rule, worked example showing correct diagram format.

### example 4: optimizing a Dockerized microservice

**Configuration:**
- Target files: `src/handlers/order.go`, `src/db/queries.go`, `docker-compose.yml`
- Output mode: single-winner / Rollback: git
- Guards:
  - `docker compose build --quiet`
  - `docker compose up -d && timeout 60 sh -c 'until docker compose exec api curl -sf http://localhost:8080/health; do sleep 1; done'` (the poll loop is itself capped — an uncapped `until` can hang the run)
  - `docker compose exec api go test ./...`
- Evaluators:
  - command: `hey -n 500 -c 20 http://localhost:8080/api/orders | awk '/Average:/{print $2*1000}'` / extract: `raw` / check: `< 50`
  - command: `hey -n 500 -c 20 http://localhost:8080/api/orders | awk '/99% in/{print $3*1000}'` / extract: `raw` / check: `< 200`
  - command: `docker stats --no-stream --format '{{.MemUsage}}' api | awk -F'MiB' 'NF>1{print $1}'` / extract: `raw` / check: `< 256` (if docker reports GiB the extraction yields nothing and the eval fails — correct, since GiB-scale usage exceeds the threshold anyway)
  - judgment: "Does the /api/orders endpoint still return correct, complete order data matching the original response schema?"
- Timeout: 600s (docker rebuild per experiment + health poll + 3 eval runs)
- Max iterations: 25
- Runs: 3

**Result:** Baseline 25% → Final 100% in 5 experiments. Key wins: connection pooling in database layer, query batching for N+1 problem, response payload trimming with field selection, enabling gzip compression in Docker reverse proxy config.

### example 5: optimizing a Go CLI tool's execution speed

**Configuration:**
- Target files: `cmd/process/main.go`, `internal/parser/parser.go`, `internal/pipeline/pipeline.go`
- Output mode: single-winner / Rollback: git
- Guards:
  - `go build ./cmd/process`
  - `go test ./...`
- Evaluators:
  - command: `hyperfine --warmup 3 --export-json /tmp/hf.json './process testdata/large.csv' && jq -r '.results[0].mean' /tmp/hf.json` / extract: `raw` / check: `< 0.5` (hyperfine has no stdout JSON mode — export to a file and read it back)
  - command: `diff <(./process testdata/large.csv) testdata/large.golden | wc -l` / extract: `raw` / check: `< 1` (output identical to the saved baseline — deterministic correctness belongs in a command eval, not a judgment)
  - command: `/usr/bin/time -l ./process testdata/large.csv 2>&1 | awk '/maximum resident/{print $1}'` / extract: `raw` / check: `< 104857600` (macOS; on Linux use `/usr/bin/time -v`, the `Maximum resident set size (kbytes)` line, and a threshold of `102400`)
  - judgment: "Is the CLI's human-readable progress/error output still clear and complete?"
- Timeout: 300s
- Max iterations: 20
- Runs: 5

Note: `hyperfine` runs the command multiple times internally and reports mean execution time in seconds. Higher runs (5) help smooth out variance.

**Result:** Baseline 40% → Final 100% in 6 experiments. Key wins: replaced encoding/json with jsoniter for parsing, added worker pool for concurrent row processing, switched from map to slice for ordered results, reduced allocations by reusing buffers in pipeline.

### example 6: cold-email copy (top-N, non-code)

**Configuration:**
- Target files: `templates/cold-email.md`
- Output mode: top-3 (user asked for "a few variants I can pick from") / Rollback: git
- Guards: `markdownlint templates/cold-email.md`
- Evaluators:
  - judgment: "Does the first sentence reference a specific time, place, person, or sensory detail?"
  - judgment: "Does the email end with exactly one specific, concrete ask?"
  - judgment: "Is the output free of phrases from the banned list: [game-changer, level up, touch base, hope this finds you well, let me know if you're interested]?"
  - command: `wc -w < templates/cold-email.md` / extract: `raw` / check: `>= 40`
  - command: `wc -w < templates/cold-email.md` / extract: `raw` / check: `<= 80`
- Timeout: 300s
- Max iterations: 25
- Runs: 5

Note: the banned-phrase list was extracted from 2 "bad" examples the user pasted during the front-door; the length range was inferred from the 3 "good" examples. The two `wc` evaluators share one command (deduplicated — it runs once per run). Per-eval pass counts go to `scores.json` for finalist selection.

**Result:** Baseline 40% → Top 3 finalists at 95–100%. Key differentiators across finalists: opening hook shape (story vs. observation vs. stat), middle-paragraph evidence density, CTA phrasing.

### example 7: financial forecast (exploration mode)

**Configuration:**
- Target files: `forecast/mrr-model.csv` (exported from the team spreadsheet)
- Output mode: exploration, N=3 (bull / base / bear — `base` is the baseline itself) / Rollback: snapshot-dir (CSV is text but not in a git repo)
- Diversity dimensions: monthly growth rate (10% relative delta); churn assumption (5% relative delta); CAC payback months (20% relative delta)
- Optimize-toward: consistency + defensibility (user chose "defensibility" when warned about the optimize-toward trap)
- Guards: `python scripts/forecast_validate.py forecast/mrr-model.csv` (checks reconciliation, no #DIV/0)
- Evaluators (all judgment; hard constraints in exploration mode, each judged blind):
  - "Do all totals reconcile? (Revenue sums to ARR, costs sum to total costs, ending cash = starting cash + net cash flow.)"
  - "Are all key assumptions within defensible market ranges (monthly churn 1–8%, CAC payback 3–24 months, monthly growth 0–30%)?"
  - "Is every key assumption paired with a source, rationale, or explicit 'same as last year' note?"
- Timeout: 300s
- Max iterations: 40
- Runs: 3 (the CSV is deterministic, but the judgments are LLM-graded — use ≥3 runs whenever judgments are in play)

**Result:** Found 3 distinct valid variants (base + 2 kept) in 14 experiments. Variant differences: base (growth 8%, churn 2.8%, CAC payback 14mo), bull (growth 14%, churn 2.2%, CAC payback 11mo), bear (growth 4%, churn 4.5%, CAC payback 20mo). All reconcile, all defensible.

### example 8: research-prompt output consistency (single-winner, non-code)

**Configuration:**
- Target files: `prompts/competitor-research.md`
- Output mode: single-winner / Rollback: git
- Guards: `echo ok`
- Test set: 3 fixed competitor names written to `autoresearch-[name]/tasks/` at setup; each run executes the prompt against them in a fresh subagent
- Evaluators (all judgment, judged blind per generated output):
  - "Does the output include each required section: [Positioning, Pricing, Key Features, Recent Moves, Risks]?"
  - "Is every factual claim paired with a source URL or a dated 'as of' marker?"
  - "Is the output under 800 words?"
  - "Is the output free of hedging phrases like 'may', 'might', 'it is possible', 'could potentially' (replace with specifics or omit)?"
- Timeout: 600s (5 runs × LLM generation time)
- Max iterations: 20
- Runs: 5 (LLM output is non-deterministic)

**Result:** Baseline 45% → Final 95% in 7 experiments. Key wins: explicit required-sections list in the prompt, "cite a source or omit" instruction, worked example showing a competitor write-up, anti-hedging rule with worked transformations.

---

## the test

A good autoresearch run:

1. **Started with a baseline** — never changed anything before measuring
2. **Used binary evals only** — no scales, no vibes — with thresholds calibrated near the baseline
3. **Changed one thing at a time** — so you know what helped
4. **Kept a complete log** — every experiment recorded in changelog
5. **Ran guards before evaluating** — broken code never reached scoring
6. **Used anchor-based rollback** — the per-run tag / latest `-good` dir / kept export id, advanced on every keep — never `HEAD~1`
7. **Ran isolated** — the user's branch and pre-run state never saw a mutation
8. **Judged blind** — no judgment eval was graded by the agent that authored the mutation
9. **Improved the score** — measurable improvement from baseline to final (or, in exploration mode, delivered N genuinely distinct valid variants)
10. **Didn't overfit** — the target got better at the actual job, not just at passing evals
11. **Ran autonomously** — didn't stop to ask permission between experiments
12. **Re-read the changelog** — didn't repeat failed experiments or forget what worked

If the target "passes" all evals but actual quality hasn't improved — the evals are bad. Go back and write better evals.
