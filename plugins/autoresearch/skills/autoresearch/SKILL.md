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
4. Keeps mutations that improve the score — and still do when re-measured fresh — and discards the rest
5. Repeats until the score ceiling is hit, max iterations reached, or the user stops it

**Output:** Optimized target files (on a dedicated `autoresearch/[name]` branch, worked on in a separate git worktree, for git runs) + `config.yaml` holding every setting of the run + `results.tsv` log + `changelog.md` of every mutation attempted + a live HTML dashboard. Depending on output mode, the deliverable is one winner, a shortlist of finalists, or a portfolio of distinct variants.

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

> *"I'll run small changes to `templates/cold-email.md` one at a time, check each against 3–5 quality rules you pick, and keep only the versions that improve the score. I work in a separate copy of your repo on its own git branch, so you can keep working in yours, and anything that makes things worse is automatically undone. You can stop anytime."*

### Pass 3 — Present the draft

Based on Pass 1, produce a full draft configuration. Every field is marked ✓ (confident) or ? (needs user input). Present it in a compact block:

```
Run name:            cold-email                                        ✓
Target files:        templates/cold-email.md                           ✓
Target type:         text / writing                                    ✓
Rollback:            git (worktree, branch autoresearch/cold-email)    ✓
Output mode:         top-3 — you asked for "options"                   ✓

Objective:           pass count of evals 1–2 (higher is better)        ✓
Evals (drafted — review, edit, remove, or add):
  1. Opening specificity (judgment)                                    ?
     — Your good examples all open with a specific time/place. Check.
  2. Single concrete ask at the end (judgment)                         ?
     — Your good examples end with a specific ask. Check.
Constraints (must pass every run, or the change is thrown out):
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

Keep asking until every `?` is resolved. When the draft is fully ✓, move to Pass 5.

### Pass 5 — Stress-test the evals

Before setup, while the user is still here, test the evals themselves. A run can only be as good as its evals, and every flaw found now saves a run spent climbing the wrong number.

1. **Calibrate every judgment eval** on the labelled examples the user gave. First write down the verdict each example should get on each eval — a good example should pass every eval; a bad example should fail the evals it was collected to illustrate — and show this small table to the user to correct. Then, for each eval and example, dispatch 3 blind judges with only the example and the question. The eval passes calibration if all 3 verdicts match the expected one on every example. On any mismatch, show the user the example, the verdicts, and the judges' reasons, propose a rewritten question, and calibrate again. After two failed rewrites, drop the eval or let the user edit it — and calibrate the user's version too: an eval that disagrees with the user's own examples optimizes toward something they don't want. No labelled examples → skip calibration and say so; the baseline (setup step 12) still checks that each judgment answers consistently.
2. **Present the results** with the final draft — calibration verdicts and any rewritten evals — and get the user's confirmation. Then move to the setup checklist below.

### Inline education rules

- First mention of a concept = ≤15-word gloss. Second mention = no gloss. Tracked per session.
- Concepts to gloss: baseline, eval, guard, mutation, rollback, score, pass rate, iteration, keep, discard, variant, diversity dimension, objective, constraint, noise margin, calibration.
- Every proposed eval has a `Grounded in:` line.
- At every step, offer two escape hatches: *"skip this, use a default"* and *"I want to edit directly"* (drops the user into the raw configuration form below).

### the configuration fields (produced by the front-door)

Pick a short kebab-case run name `[name]` (e.g. `nextjs-perf`, `cold-email`) — it names the branch, the rollback anchor, and the artifacts directory. The front-door must populate all of the following. Power users can say "I want to edit directly" and fill them in raw.

1. **Target files** — Explicit list of file paths (relative to the repo root) the agent can edit. Nothing else is editable. In git runs, edit them only at their path inside the worktree, `<worktree>/<path>` — never the copy in the user's checkout.

2. **Evaluators** — List of checks (3-6 recommended), drawn from [references/eval-patterns.md](references/eval-patterns.md) and grounded in user examples where possible (see [references/eval-guide.md](references/eval-guide.md) for principles). Each check is a command evaluator or a judgment evaluator (formats below) and has one of two **roles**:

   - **Objective** — what the run improves. Exactly one per run, of one of two kinds:
     - **metric** — a single command evaluator with a `direction` (`lower` or `higher`) instead of a `check`; its raw number is compared against the current best. Use it whenever the quality is a number: latency, runtime, bundle size, a Lighthouse score. An optional `target` value stops the run once reached. Ask the user for the smallest change that matters to them (default: the measured noise).
     - **pass-count** — the total passes of the scored checks across all runs (see "scoring"). Use it when quality is a set of yes/no properties: copy, prompts, documents.
   - **Constraint** — a check that must pass in every run, or the candidate is discarded whatever its objective. Use it for what must not break: output identical to a golden file, required content still present, no banned phrases. Pair every metric objective with at least one correctness constraint — speed is easy if you're allowed to delete features. A constraint must be stable: one that flips on the unchanged baseline (setup step 12) would discard good changes at random.

   A configuration that assigns no roles — every check scored, no constraints — is a pass-count objective over all checks and behaves as runs always have.

   **Command evaluator** — a shell command, extraction path, and threshold (a metric objective has a direction instead):
   ```
   command:   "npx lighthouse http://localhost:3000 --output=json --quiet"
   extract:   ".categories.performance.score"
   check:     ">= 0.9"        # scored checks and constraints
   direction: higher          # metric objective, instead of check
   ```
   Extraction formats: jq-style path for JSON (`.field.subfield`), `"field N"` for whitespace-delimited output, `"raw"` for plain numeric output. A check can also be `exit 0` with no `extract`: it passes when the command exits 0 (a missing file or crash is then a failure, not a silent pass).

   **Calibrate thresholds against the current baseline.** When the quality is a number, make it the metric objective and skip thresholds altogether. For thresholded scored checks, binary scoring is blind to sub-threshold movement: if the baseline is 0.62 and the threshold is `>= 0.9`, an improvement to 0.85 scores exactly the same as no improvement and gets discarded. Either set thresholds just beyond the current value so progress can register, or use stepped thresholds — the same command as three evaluators with `>= 0.7`, `>= 0.8`, `>= 0.9` — so each increment flips an eval.

   **Judgment evaluator** — a binary yes/no question, answerable from the artifact alone: no "still", "better than before", or other reference to a version the judge never sees. Three things must be pinned down at setup:
   - **What is judged and how it is produced fresh each run.** For code: fetch the page, run the binary, read the build artifact. For skill/prompt targets: write a fixed set of 3-5 test prompts into `autoresearch-[name]/tasks/` at setup; each run executes the target against them in a fresh subagent, loading it from its path inside the worktree (`<worktree>/<path>`) — an installed or project copy of a skill, prompt, or config is the user's unedited version and would measure nothing. A judgment with no fresh artifact to inspect is ungrounded — the score would just measure the agent's optimism about its own edit.
   - **Who judges: a fresh subagent, blind to the experiment.** Dispatch a subagent that receives ONLY the artifact, the yes/no question, and any reference file the question names (an inventory, a schema) — never the diff, the hypothesis, or the changelog. The agent that authored a mutation must not grade it; self-graded judgments say "yes" almost every time. The judge answers `PASS` or `FAIL` plus one sentence saying why; the reasons feed the next experiment's analysis.
   - **That it agrees with the user.** Calibrated on their labelled examples in Pass 5.

   Both types can be mixed in a single run. Prefer command evaluators wherever the quality is mechanically checkable (a deterministic "is the output identical" check belongs in a `diff`-based command eval, not a judgment).

   **Command deduplication:** When multiple evaluators share the same command string, the command runs once per run. All evaluators sharing that command parse the same output.

3. **Guards** — Shell commands that must exit 0 after every mutation (at least one required). If any guard fails, the mutation is auto-discarded without running evaluators.
   - Code: `npm run build`, `npm test`, `pytest tests/`
   - Non-code targets without a natural build step: `echo ok`, or a structural check like `markdownlint`
   - **Whatever an evaluator measures must come from the worktree.** If an evaluator hits a server or container, a guard (re)starts it from the worktree after every mutation, on a port of its own (docker compose: a project name of its own, `-p autoresearch-[name]`, and host ports the user's own stack doesn't use). Never measure a server the user is running — it serves their checkout, not the mutation.
   - Guards must not modify tracked files: use the check-only form of formatters and linters (`--check`, not `--fix`/`--write`), and no lockfile-rewriting installs.

4. **Timeout** — Max seconds per experiment (required). The budget covers one full experiment: all guard commands plus all N evaluation runs combined. If exceeded, the experiment is auto-discarded. See "enforcing the timeout" below. Rule of thumb: `guard time + runs × (sum of unique evaluator command times) + 20% margin`.
   - Lighthouse, 3 runs: ~300s
   - API benchmarking, 3 runs: ~240s
   - Docker rebuild + integration tests: ~600s
   - Skill/prompt optimization, 5 LLM runs: ~600s

5. **Max iterations** — Max experiment cycles before stopping (required); the baseline, experiment 0, and any re-baseline don't count. Forces you to choose a compute budget. Can be set high (100) but must be explicit.

6. **Runs per experiment** — How many times to evaluate per mutation. Defaults to 5 if unspecified. 3 is fine for deterministic benchmarks. 5 for nondeterministic outputs (skill prompts, LLM judgments).

Plus two front-door outputs:

7. **Output mode** — single-winner / top-N / exploration, per [references/output-modes.md](references/output-modes.md). Exploration additionally needs 1–3 diversity dimensions with thresholds (no dimension nameable → run top-N instead).

8. **Rollback mechanism** — git / snapshot-dir / API / manual-confirm, per [references/rollback-mechanisms.md](references/rollback-mechanisms.md). One mechanism per run, covering every target.

### scoring

Every check is binary — pass or fail — except a metric objective, which is a raw number. Command evaluators extract a value and check it against the threshold. Judgment evaluators are yes/no.

**Constraints** are checked in every run. A single failure in any run discards the candidate; constraints never add to the score.

**Pass-count objective:**
**Total score** = passes across all scored checks × all runs.
**Max score** = number of scored checks × runs per experiment.
**Pass rate** = score / max_score.

**Metric objective:** the value is the mean of the extracted number across the runs.

Each "run" executes all unique commands once (deduplicated), then scores all evaluators against the output. So 4 evaluators using 2 unique commands with 3 runs = 6 command executions total, 12 pass/fail scores. Max score = 12.

**Noise margin** — the minimum improvement that counts, measured on the unchanged baseline at setup (step 12) and recorded in `config.yaml`:
- pass-count: the spread (max − min) of the three baseline totals, and at least 2 if any scored check is a judgment or otherwise non-deterministic — a +1 blip on a noisy eval is indistinguishable from variance. With only deterministic checks, any strict improvement counts (margin 1).
- metric: the spread of the three baseline means (any strict improvement if the spread is zero, e.g. a deterministic size), or the user's minimum meaningful change if that is larger.

**Failed commands:** if a command exits non-zero, is killed by the timeout, or its extraction yields no numeric value, every check bound to that command scores fail for that run (an `exit 0` check simply fails on a non-zero exit). A metric objective with no value in any run leaves the experiment unmeasurable: discard it. Never substitute a stale or guessed value.

**Experiments rejected before scoring** (guard failure or timeout) are never evaluated: log them with empty score/max_score/pass_rate/objective fields in `results.tsv` and `null` pass_rate in the dashboard data. Do not fabricate a score for an experiment that was never measured.

In **exploration mode**, every check is a hard constraint and there is no objective — a candidate must pass all of them to be kept, and score is replaced by a distinctness check against existing kept variants. See [references/output-modes.md](references/output-modes.md).

---

## setup

Once the front-door has confirmed all fields, run these steps in order. The git mechanism is shown inline (the common case); for snapshot-dir, API, and manual-confirm the same steps dispatch to the per-mechanism operations in [references/rollback-mechanisms.md](references/rollback-mechanisms.md).

1. **Confirm configuration** — all fields populated from the front-door: target files, evaluators with their roles, guards, timeout, max iterations, runs per experiment, output mode, rollback mechanism, run name. A field that breaks a rule above — a number checked against a far-off threshold, a judgment that says "still" — goes back to the user before anything else runs.
2. **Safety-review guards and evaluators.** These commands run unattended, dozens of times. Refuse or explicitly confirm anything destructive or outward-facing: deletes outside the artifacts dir, `sudo`, piping downloads to a shell, benchmarks pointed at production URLs. Also check that no target file is an eval input — a test the guards or evaluators run, a golden file, a task file. A target the run may edit can't also be what measures it: split the file or drop it from the evals.
3. **Run the rollback pre-flight** per [references/rollback-mechanisms.md](references/rollback-mechanisms.md). For git that means: resolve the target repo (`git -C <dir-of-target> rev-parse --show-toplevel` — ALL targets must resolve to the SAME repo); every target file is tracked (`git ls-files --error-unmatch <path>`) and has no uncommitted changes (abort otherwise — do NOT auto-commit the user's work; other uncommitted or untracked files don't block the run, but tell the user they aren't part of it); no collision with a previous run's tag, branch, worktree, or artifacts dir (offer resume or a new name — never overwrite a previous run's `baselines/`). If any pre-flight fails, abort with that doc's message.
4. **Read and understand all target files.** For code: architecture, dependencies, what each file does. For writing: tone, structure, intended audience. For forecasts: assumption cells, formula chains, which cells feed the outputs the user cares about.
5. **Create the run worktree** (git): `git -C <target repo> worktree add -b autoresearch/[name] <worktree> HEAD`, with `<worktree>` defaulting to `<parent of target repo>/<repo dir name>-autoresearch-[name]`. Every mutation, guard, and evaluator runs inside the worktree — git commands as `git -C <worktree>`, shell commands with the worktree as working directory. The user's checkout, branch, and uncommitted work are never touched, so they can keep working while the run goes.

    **Prepare the worktree.** It holds tracked files only — no `node_modules`, `.venv`, build caches, or ignored config like `.env`. Run the project's install step in it (`npm ci`, `uv sync`, `bundle install`, `go mod download`, …) and record those commands under `worktree_setup` in `config.yaml`, so recovery can repeat them. Install into the worktree's own environment (a `.venv` inside it, its own `node_modules`) and run guards and evaluators with that one; an editable install or `npm link` made from the user's environment points at their checkout, so every candidate would measure the unmutated code. Never copy a secret file (`.env*`, credentials) into the worktree without asking the user.
6. **Verify guards pass** — write each guard and evaluator command into `autoresearch-[name]/commands/<name>.sh` first (step 11 describes the scripts), then run them — in the worktree (git) or on the current state (other mechanisms), each wrapped in `timeout` within one experiment's budget — the same budget each of the three baseline evaluations in step 12 gets. If any fails, stop and tell the user — the target is already broken, or (git) a guard depends on uncommitted or ignored files that are not in the worktree; say which. Then `git -C <worktree> status --porcelain --untracked-files=no` must be empty: a guard that rewrites tracked files (a formatter's `--fix`, code generation, a lockfile update) would make every experiment look like it touched non-target files — have the user switch it to its check-only form.
7. **Create the artifacts directory** `autoresearch-[name]/`. Git: at the root of the user's checkout — outside the worktree, so rollbacks never touch it and it outlives the worktree — and add `/autoresearch-[name]/` to the repo's local exclude file if absent, which keeps it out of the user's `git status` without committing anything. Get its absolute path with `git -C <target repo> rev-parse --path-format=absolute --git-path info/exclude` (the plain `--git-path` form prints a path relative to the repo, not to your working directory) and `mkdir -p` its directory first. Tell the user not to run `git clean -x` in their checkout during the run: it deletes ignored directories, this one included. Remove the exclude line when the run is delivered. Other mechanisms: in the targets' common directory.
8. **Back up all target files** to `autoresearch-[name]/baselines/`, mirroring their repo-relative paths. These are the pre-run originals — an abort escape hatch, not the rollback mechanism, and identical across all mechanisms.
9. **Establish the rollback anchor.** Git: `git -C <worktree> tag autoresearch/[name]/good autoresearch/[name]`. Snapshot-dir: copy targets to `iterations/0000-baseline/` and `0000-good/`. API: export and record iteration 0. Manual-confirm: log the baseline and get the user's ack.
10. **Write `autoresearch-[name]/config.yaml`** — every confirmed field and the run's bookkeeping (format below). Filling in what setup measures afterwards — checksums, noise margins, baseline values — completes revision 1 rather than changing it. It is the single source of truth for resuming: recovery reads it first, and nothing a resume needs may exist only in the conversation. Evaluator wording, commands, thresholds, guards, and the judge model are frozen once the baseline has run. Changing any of them mid-run means bumping `revision` and re-running the baseline, logged as a new `baseline` row; from then on compare only against scores measured under the new revision.
11. **Create artifacts:** `results.tsv` (with header row), `changelog.md` (empty), `dashboard.html` (copy from [references/dashboard-template.html](references/dashboard-template.html), replace `__DATA_PLACEHOLDER__` with initial data JSON including the `mode` field), `scores.json`, and — for targets evaluated over a set of inputs — the fixed test-task set under `autoresearch-[name]/tasks/`. Save the labelled examples from Pass 5 to `autoresearch-[name]/examples/`. Write every guard and evaluator command — including the ones that produce a judgment's artifact — verbatim from `config.yaml` into its own script, `autoresearch-[name]/commands/<name>.sh`, and run it from then on as `timeout <remaining>s bash <script>` from the worktree root — no shell quoting to get wrong. Record the sha256 of every file — not directory — among the eval inputs outside the worktree (everything under `commands/`, `tasks/`, `examples/`, and any golden file kept in the artifacts directory), and re-verify them before the baseline under `eval_inputs_sha256` in `config.yaml`. Open the dashboard: `open autoresearch-[name]/dashboard.html` (macOS) / `xdg-open` (Linux).
12. **Run the baseline** (experiment 0) — evaluate the unchanged state three times over (three full evaluations of all checks × all runs). This doubles as the noise measurement:
    - **Per check:** stable pass, stable fail, or flips. A constraint that flips can't stay a constraint: stop before any experiment and tell the user, with the flipping verdicts and judge reasons — rewrite it to be deterministic or, in a pass-count run, move it into the score (in a metric run, rewriting is the only option). A constraint that fails every time on the unchanged baseline would discard every candidate: stop and tell the user, who can fix the target first or change the constraint. A scored check that passes every time carries no signal: report it, and drop or tighten it if the user is around; otherwise keep it — it dilutes the pass rate but can't distort a comparison.
    - **Margins:** compute the objective's noise margin as defined in "scoring" and record it under `noise:` in `config.yaml`.
    - **Baseline score:** the median of the three pass-count totals, or the mean of the three metric means. This is the anchor's stored score and the value logged in row 0.
    - **A baseline evaluation that times out** means the experiment timeout is too small for these evals: stop and ask the user to raise it — every experiment would time out the same way.

    Then (exploration: none of this — the baseline passes everything by design, there is no objective or noise margin, and the run proceeds):
    - 100% (pass-count), or the metric `target` already met → inform the user, stop.
    - Within reach of the ceiling — max_score − 1, or so close that no candidate could clear the noise margin (max_score − score < margin) → warn the user and ask whether it's worth proceeding.
    - Otherwise → report the baseline score, the margins, and any flagged checks, and proceed. Do not wait for an acknowledgment — the user may already be away.

    In **exploration mode**, the baseline must pass ALL evals in all three evaluations — it becomes variant `base` and counts toward N. If it fails any eval, abort: exploration requires a valid starting point.

### config.yaml

Written at setup step 10 and kept current: rewrite it whenever the run's settings change (with a `revision` bump). Example for a git run:

```yaml
name: fast-parse
revision: 1                       # bump on any evaluator/guard/threshold change, then re-run the baseline
mode: single                      # single | top-n | exploration
n: null                           # top-n / exploration: the requested N
diversity_dimensions: []          # exploration: [{name: growth rate, threshold: "10% relative"}]
rollback: git                     # git | snapshot-dir | api | manual-confirm
repo: /work/proc
base: {branch: feature/x, commit: 1a2b3c4}   # where the run started; integration targets this branch
worktree: /work/proc-autoresearch-fast-parse
run_branch: autoresearch/fast-parse
anchor_tag: autoresearch/fast-parse/good
artifacts: /work/proc/autoresearch-fast-parse
targets: [cmd/process/main.go, internal/parser/parser.go]
worktree_setup: ["go mod download"]  # run in a fresh worktree, again on recovery
guards: ["go build ./cmd/process", "go test ./..."]  # each also written to commands/<name>.sh
timeout_s: 300
max_iterations: 20
runs: 5
objective:
  kind: metric                    # metric | pass-count
  name: mean runtime
  command: "hyperfine --warmup 3 --export-json /tmp/hf.json './process testdata/large.csv' && jq -r '.results[0].mean' /tmp/hf.json"
  extract: raw
  direction: lower
  unit: s
  target: 0.5                     # optional — stop once the anchor reaches it
constraints:
  - name: output identical
    type: command
    command: "diff -q <(./process testdata/large.csv) testdata/large.golden"
    check: "exit 0"
  - name: progress output clear
    type: judgment
    question: "Does the progress output show a percentage, the current row count, and an ETA on every update line?"
    artifact: "stderr of ./process testdata/large.csv"
judge_model: claude-opus-5-5      # the model the judges actually ran on at baseline — a resume on another model means re-baselining
calibration:
  progress output clear: "3/3 judges gave the expected verdict on all 3 labelled examples"
noise: {objective_margin: 0.07}
baseline: {objective: 1.81}
eval_inputs_sha256:               # eval inputs outside the worktree; checked before every evaluation
  commands/mean-runtime.sh: 4b1e…
  examples/progress-good-1.txt: 9f2c…
```

A pass-count run has `objective: {kind: pass-count, checks: [...]}` listing the scored checks in the same format as constraints. Non-git runs omit `repo`, `base`, `worktree`, `run_branch`, and `anchor_tag`.

---

## the experiment loop

Once the baseline is reported, the loop runs autonomously. Do not pause to ask the user between experiments. (Exception: the manual-confirm rollback mechanism pauses for the user's snapshot ack each iteration — there, the pause IS the mechanism.)

**LOOP:**

**0. Re-read** — Read `config.yaml`, `results.tsv`, and the last 10 entries of `changelog.md`. Skip on first iteration. This keeps the run's settings and experimental context alive across context window compression.

**1. Analyze** — Which evals fail most? Read the actual outputs or command results that failed, and the judges' one-line reasons. Identify the pattern: is it a formatting issue? A performance bottleneck? A missing optimization? An ambiguous instruction?

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

**3. Verify the anchor** — Confirm the rollback anchor matches the current target state (git: the worktree is on the run branch — `git -C <worktree> symbolic-ref --short HEAD` prints `autoresearch/[name]` — and `autoresearch/[name]/good` points at its HEAD). It always should: KEEP advances it and DISCARD resets to it. If they diverge, something went wrong — stop and run recovery instead of force-tagging over the discrepancy. (Exploration: the anchor is the base state — every candidate starts from it.)

**4. Mutate** — Edit target file(s) — at `<worktree>/<path>`, for git runs — then record the attempt per the rollback mechanism. Git: stage ONLY the declared target files (`git -C <worktree> add <each target path>` — never `git add -A`; sweeping in unrelated files is how a later discard deletes them) and commit as a single commit with `--no-verify` (a pre-commit hook that rewrites files would silently change the mutation; one that rejects the message format would derail the loop). Message format: `autoresearch: [short description of change]`. If git finds nothing to commit, the edit landed somewhere else — most likely in the user's checkout: stop the run and tell the user which file; don't undo anything in their checkout yourself. Snapshot-dir: copy the mutated targets to `iterations/NNNN-attempt/`.

**Stay inside the targets.** Git: record `git -C <worktree> status --porcelain --untracked-files=all` before editing; after the commit it must be identical, and `git -C <worktree> show --stat --format= HEAD` must list only target files. Any difference — a modified tracked file, a new file — means the mutation reached outside the targets: an edited test, a rewritten golden file, a new source file the build picks up. Roll back to the anchor, delete the new files, log `discard` ("touched non-target files"), and go to step 0. Ignored paths (`node_modules/`, build output) don't appear in status: keep eval inputs tracked or in the artifacts directory (checksummed), never in an ignored path. The evals measure the targets only as long as nothing else changes.

**5. Guard** — Run all guard commands, each wrapped in `timeout <remaining>s` (see "enforcing the timeout").
- Any guard fails → roll back to the anchor (git: `git -C <worktree> reset --hard autoresearch/[name]/good`), log as `guard_fail`, go to step 0.
- Budget exceeded or a command killed (exit 124) → roll back the same way, log as `timeout`, go to step 0.
- Git: afterwards `git -C <worktree> status --porcelain --untracked-files=no` must still be empty. A guard that rewrote a tracked file would put an unmeasured change under the candidate → roll back, log `discard` ("guards modified tracked files"), go to step 0.

**6. Evaluate** — First verify the eval inputs: recompute the sha256 of every file under `eval_inputs_sha256` in `config.yaml` — here and before every other evaluation in this experiment, re-measurement included. A mismatch means an eval input changed during the run — stop the run, set the dashboard `status` to `"error"`, and tell the user; scores from here on would not be comparable. Then run all evaluators × N runs, every command timeout-wrapped, tracking the remaining experiment budget. For each run: execute all unique commands once (deduplicated), then score every evaluator against the output. For judgment evaluators: produce the fresh artifact, then dispatch a blind subagent with only the artifact and the question; it returns `PASS`/`FAIL` and one sentence why. If the budget runs out mid-evaluation, treat the experiment as `timeout` (roll back, log, go to step 0). Afterwards the tracked-files check from step 5 applies again — after this and every later guard run and evaluation in the experiment, re-measurement and reproduction included: an evaluator that rewrote a tracked file means the measurement isn't of the committed candidate.

**7. Decide** — by output mode:

- **single-winner** — work through these in order; the first DISCARD ends the experiment (roll back to the anchor):
  1. **Constraints.** Any constraint failed in any run → **DISCARD.**
  2. **Objective.** Compare with the anchor's stored score. Improved by at least the noise margin → a would-be keep. Not worse than the anchor's stored score, and the change strictly simplifies the target files (net-negative diff) → a would-be simplification keep; note it as such in the description. Otherwise → **DISCARD.**
  3. **Re-measure.** A would-be keep was picked on the very measurement that made it look good; keeping it on that draw ratchets noise into the anchor. Evaluate the anchor and the candidate again, fresh, constraints included: switch the target to the anchor state, run the guards (which rebuild it), evaluate; switch back to the candidate, run the guards, evaluate. For a metric objective, repeat the pair once more in alternation (anchor, candidate, anchor, candidate), so drift on the machine hits both, and compare the mean of the candidate's two fresh values with the mean of the anchor's two. Each of these evaluations gets a fresh timeout budget. Decide on the fresh numbers only: the candidate must still beat the fresh anchor by at least the margin and pass every constraint; a simplification — including an improvement that no longer shows but whose diff is net-negative — must be no worse than the fresh anchor. Otherwise → **DISCARD** ("did not hold on re-measurement"). Whatever the outcome, end back on the candidate on the run branch (git: `git -C <worktree> checkout autoresearch/[name]`) before anything else happens. Switching per mechanism: [references/rollback-mechanisms.md](references/rollback-mechanisms.md); in manual-confirm runs the user makes each switch, as for a rollback.
  4. → **KEEP.** **Immediately advance the anchor** to the run branch's tip (git: `git -C <worktree> tag -f autoresearch/[name]/good autoresearch/[name]` — name the branch, never rely on HEAD), then log it (step 8). The anchor's stored score — the score on the latest `keep` or `baseline` row of `results.tsv` — is the candidate's fresh re-measured score, never the selection-round score. Only a KEEP or a re-baseline changes it; fresh anchor measurements taken during a discarded experiment don't. The anchor must always point at the current baseline — if the run stops right after a keep, a stale anchor makes recovery silently destroy the best result.

- **top-N:** identical to single-winner during the loop. Selection happens after the loop stops, per [references/output-modes.md](references/output-modes.md), from the per-eval results in `scores.json`.

- **exploration:**
  - All guards pass AND all evals pass — in the evaluation and again in one fresh re-evaluation — AND the candidate differs from every existing kept variant on at least one diversity dimension → **KEEP as a new variant**: record its anchor at the branch tip (git: `git -C <worktree> tag autoresearch/[name]/variant-K autoresearch/[name]`), log it, then return the targets to the base state (git: `git -C <worktree> reset --hard autoresearch/[name]/good`) so the next candidate also mutates from base.
  - Any eval fails OR not distinct → **DISCARD.** Roll back to the base anchor.
  - "Score improvement" is not the decision rule — validity plus distinctness is.

**8. Log** — Append to `results.tsv` (exploration keeps use a `variant K — …` description); every scored row logs the experiment's latest measurement — for keeps, and for any candidate that reached re-measurement, the fresh one. Append the experiment's per-eval pass counts and objective value to `scores.json` (every mode). Update `dashboard.html` with new inline data. Append to `changelog.md` (see format below).

**9. Check stop conditions:**

- User manually stops → stop, deliver results.
- Max iterations reached → stop, deliver results.
- **single-winner / top-N — ceiling reached:** pass-count: the standing baseline score is ≥ max_score − 1, or so close to max_score that no candidate could clear the noise margin (max_score − score < margin), AND the last 3 consecutive experiments (kept or discarded) failed to improve it → stop, deliver results. Judge this against the baseline's own score — never against the logged score of a discarded mutation, which is lower by definition. Metric: the anchor's stored value has reached the objective's `target`, if one is set → stop, deliver results.
- **exploration:** N distinct valid variants found (counting `base`) → stop, deliver results.
- (Per-experiment timeout is handled in steps 5-6 — that experiment is discarded, loop continues.)
- Otherwise → go to step 0.

**NEVER STOP between experiments to ask the user.** They may be away. Run autonomously until a stop condition is met or the user interrupts.

**If you run out of ideas:** Re-read the failing outputs. Re-read the changelog. Try combining two previous near-miss mutations. Try a completely different approach to the same problem. Try removing things instead of adding them — a simplification that holds the score is a keepable win.

### enforcing the timeout

An agent cannot interrupt a hung shell command, so the timeout only exists if every command is wrapped. Record the experiment start time. Before each guard or evaluator command, compute the remaining budget and prefix the command with `timeout <remaining>s` (GNU coreutils; on macOS `brew install coreutils` provides it as `timeout` or `gtimeout`). Every guard and evaluator lives in its own script under `commands/` (setup step 11), so it is wrapped whole — `timeout <remaining>s bash commands/<name>.sh`, run from the worktree root — however many pipes, `&&`s, quotes, or process substitutions it contains. Never inline a command as `bash -c '…'`: its own single quotes end the string early. Exit code 124 means the command was killed — treat it as the budget being exceeded. Never run a guard or evaluator bare: a mutation can pass every guard and still hang at runtime (an accidental infinite loop passes `py_compile` and unit tests that don't call the hot path), and an unwrapped evaluator would block the loop forever. Judge subagents cannot be wrapped; count their wall time against the same budget, and if the budget is gone once they return, the experiment is a `timeout`.

---

## rollback strategy

Full per-mechanism details in [references/rollback-mechanisms.md](references/rollback-mechanisms.md). Principles that hold for every mechanism:

- **One change per experiment.** All target-file edits land in a single commit (git) / a single snapshot dir (snapshot-dir) / a single export (API) / a single user-confirmed change (manual-confirm).
- **The anchor lives forward.** After every KEEP, advance the anchor — the `autoresearch/[name]/good` tag, the latest `-good` dir, the most recent kept `export_id`. Never depend on `HEAD~1` or similar relative references.
- **Rollback is atomic per experiment.** A discard returns ALL target files to the anchor state in one operation — a mechanism that can only revert some of the targets is the wrong mechanism for the run.
- **Clean targets required at setup.** Pre-flight checks must pass before the loop starts.
- **Reset only the run's own copy.** Git: before every `reset --hard`, confirm `<worktree>` is the run's linked worktree on the run branch — `git -C <worktree> rev-parse --absolute-git-dir` differs from `git -C <worktree> rev-parse --path-format=absolute --git-common-dir`, and either `git -C <worktree> symbolic-ref --short HEAD` prints `autoresearch/[name]`, or — only mid re-measurement or reproduction — it is detached at the anchor or base commit. A rollback or keep always happens on the branch: switch back first. Anything else means the paths are wrong: stop the run. A reset aimed at the user's checkout would destroy their uncommitted work.

For git specifically: a separate worktree on a dedicated run branch (`autoresearch/[name]`), so the user's checkout and branch never see a mutation, a reset, or a commit; per-run namespaced tag; stage only the declared target files; in-run git commands run against the worktree (`git -C <worktree> …`), repo-level ones against the resolved target repo (`git -C <target repo> …`) — neither is necessarily the cwd's repo. Commit/log message format for git and any log-based mechanism: `autoresearch: [short description]`.

### recovery

If the session ends mid-experiment (crash, disconnect, context limit):

1. **Read `autoresearch-[name]/config.yaml`.** It holds every setting the loop needs — targets, evaluators, guards, timeout, runs, mode, rollback mechanism, worktree, tag, base branch. Resume from it, never from memory. If `config.yaml` is missing but the worktree and tag exist (setup crashed after step 5 and before step 10): the run never started — clean up completely and start setup over: `git -C <target repo> worktree remove <worktree>`, delete the tag and the branch `autoresearch/[name]`, delete `autoresearch-[name]/` (its `baselines/` are copies of unmodified files), and remove the run's line from the local exclude file. Also: if `config.yaml` exists but has no `baseline` and `results.tsv` has no `baseline` row, setup stopped before the baseline — resume it at step 11. A run started before `config.yaml` existed (skill version 1.2 or earlier) has none either:
   - Rebuild the configuration from `baselines/`, the dashboard data, and the changelog, show it to the user in the Pass 3 format with every unrecoverable field marked `?`, and get it confirmed before continuing. If any evaluator could not be recovered verbatim, re-run the baseline under the confirmed configuration before comparing scores. (Mechanism: `api-state.json` → API, `manual-snapshots.md` → manual-confirm, `iterations/` → snapshot-dir, otherwise git.)
   - Git: such a run worked in place — its run branch is checked out in the user's own checkout. Never reset that checkout. Tell the user, and offer to continue once they have switched their checkout back to their own branch (committing or stashing their work is their call); then attach a worktree to `autoresearch/[name]` and continue under these rules.
   - Migrate `results.tsv`: if its header lacks `objective`, keep a copy as `results.tsv.bak`, then append the empty `objective` column to its header and every existing row, so new rows line up.
2. **Return to the anchor.** Git:
   - If the worktree directory is missing, clear only its stale entry — `git -C <target repo> worktree remove <worktree>` (not `worktree prune`, which acts on every worktree of the repo) — re-attach it with `git -C <target repo> worktree add <worktree> autoresearch/[name]`, and re-run the `worktree_setup` commands.
   - If the worktree is on a detached HEAD — the crash came mid re-measurement — check out the run branch first (`git -C <worktree> checkout autoresearch/[name]`; if a modified tracked file blocks the checkout, `git -C <worktree> reset --hard` on the detached state first).
   - A crash between moving the anchor tag and logging a keep leaves the tag one commit ahead of the log. The tag always wins — never move it backwards. If the tag's commit isn't the one the latest `keep` row was logged for (none yet → `base.commit`), the keep was decided but never logged: re-run the baseline on the anchor and log it as a new `baseline` row, which is also the stored score from then on. Exploration: the `good` tag stays on the base commit by design, and each kept variant has its own `variant-K` tag; a `variant-K` tag with no log row is logged the same way, and the branch is reset to `good`.
   - Check that the session's model matches `judge_model`; if not, re-run the baseline before comparing judgment scores.
   - Then compare the worktree with the anchor including commits: if `git -C <worktree> rev-parse HEAD` differs from `git -C <worktree> rev-parse autoresearch/[name]/good`, or `git -C <worktree> status --porcelain --untracked-files=no` is non-empty, run `git -C <worktree> reset --hard autoresearch/[name]/good` (after the checks in "Reset only the run's own copy"). A clean working tree is not enough — the interrupted experiment's mutation is usually already committed. Other mechanisms: their "Resume after crash" in [references/rollback-mechanisms.md](references/rollback-mechanisms.md). Because the anchor advances on every KEEP, this restores the latest kept state and never loses kept work.
3. **The interrupted experiment was never scored.** Write no `results.tsv` row for it, append `## Experiment [N] — interrupted, rolled back` to the changelog, and reuse its number.
4. `baselines/` holds the PRE-RUN originals. Restoring from baselines is a full abort that discards every kept improvement — only do it if the user explicitly wants to abandon the run, and say so when offering it.
5. If abandoning, set the dashboard `status` to `"error"` so the auto-refreshing page stops claiming the run is live.
6. To resume: re-read `results.tsv` and `changelog.md`, then re-enter the loop at step 0.

---

## changelog format

After every experiment (kept or discarded), append to `changelog.md`:

```markdown
## Experiment [N] — [baseline/keep/discard/guard_fail/timeout/interrupted]

**Score:** [X]/[max] ([percent]%), or the objective value for metric runs  (or "not scored" for guard_fail/timeout)
**Change:** [One sentence describing what was changed]
**Reasoning:** [Why this change was expected to help]
**Result:** [What actually happened — which evals improved/declined, any constraint that failed]
**Re-measured:** [Fresh anchor vs fresh candidate, if the experiment got that far]
**Failing outputs:** [Brief description of what still fails, with the judges' reasons, if anything — dev tasks only]
```

This changelog is the most valuable artifact. It's a research log that persists WHY things worked or failed. The agent re-reads the last 10 entries at the start of each loop iteration. Keep descriptions free of tabs and newlines — they also go into `results.tsv`.

---

## artifacts

All artifacts live in `autoresearch-[name]/` at the root of the user's checkout for git runs — outside the worktree, excluded via `info/exclude` — or the targets' common directory for non-git runs:

```
autoresearch-[name]/
├── config.yaml             # every setting of the run — read first on resume
├── dashboard.html          # live browser dashboard (auto-refreshes, data inlined)
├── results.tsv             # score log for every experiment
├── changelog.md            # mutation log with reasoning (WHY things worked/failed)
├── scores.json             # per-eval pass counts and objective value per experiment
├── tasks/                  # fixed test inputs for the evals (targets evaluated over an input set)
├── examples/               # the user's labelled examples, used to calibrate the judges
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
  "objective": {"name": "mean runtime", "unit": "s", "direction": "lower"},  // metric runs only
  "baseline_value": 1.81,         // metric runs only: objective value of experiment 0
  "best_value": 0.42,             // metric runs only: best value among baseline + kept experiments
  "variants_target": 3,           // exploration only: the requested N
  "variants": [                   // exploration only: gallery data
    {"id": "base", "label": "base", "dimensions": {"growth_rate": "8%"}, "evals_passed": 3, "evals_total": 3}
  ],
  "experiments": [
    {
      "id": 0,
      "score": 4,                  // raw pass count
      "max_score": 12,
      "pass_rate": 33.0,           // percent; null for guard_fail/timeout rows and metric runs
      "objective_value": null,     // metric runs only; null when not measured
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

For metric runs, leave `baseline_score`, `best_score`, `score`, `max_score`, and `pass_rate` null and fill the `objective` fields; the chart then plots the objective value. `eval_breakdown` lists the constraints and scored checks from the most recent full evaluation of the current baseline — not a cumulative tally across all experiments. The experiments table renders in every mode. When the loop stops, set `status` to `"complete"` (or `"error"` if the run was abandoned) so the dashboard shows a finished state.

### results.tsv

Tab-separated with columns: experiment, score, max_score, pass_rate, status, description, objective (the newest column goes last, so older readers and scripts keep working). A row may end before `objective` — a missing trailing field means empty.

`score`/`max_score`/`pass_rate` carry a pass-count objective; `objective` carries a metric objective's value (leave whichever doesn't apply empty).

Status values: `baseline`, `keep`, `discard`, `guard_fail`, `timeout`. Guard failures and timeouts were never scored — leave their score, max_score, pass_rate, and objective fields empty. Exploration keeps prefix the description with `variant K —`.

```
experiment	score	max_score	pass_rate	status	description	objective
0	3	9	33.3%	baseline	original code — no changes
1	4	9	44.4%	keep	dynamic import for Hero component
2	4	9	44.4%	discard	added priority to hero image — no change
3				guard_fail	aggressive tree shaking — build broke
4				timeout	full image optimization pipeline — exceeded budget
5	6	9	66.7%	discard	inline critical CSS — did not hold on re-measurement (fresh 5 vs anchor 4)
6	7	9	77.8%	keep	CSS modules + font optimization
```

---

## deliver results

When the loop stops (max iterations, ceiling hit, variants found, or user interrupts), present:

1. **Score summary:** Baseline score → Final score (% improvement), or baseline → final objective value for metric runs. Exploration: variants found of N requested.
2. **Total experiments run:** kept / discarded (and how many of those failed only on re-measurement) / guard failures / timeouts
3. **Top 3 most impactful changes** (from the changelog)
4. **Remaining failure patterns** (what still fails, if anything)
5. **Mode-specific deliverable:** top-N — the finalist table (per-eval score vectors from `scores.json`) and the "which one?" prompt; exploration — the variant gallery with per-variant "what's different" summaries. See [references/output-modes.md](references/output-modes.md) for selection and materialization mechanics.
6. **Location of all artifacts**
7. **Integration (git):** the optimized state lives on the `autoresearch/[name]` branch, checked out in the worktree. After any finalist/variant selection is materialized (the variant tags are what materialization restores from — don't delete them before the user has picked), delete the `autoresearch/[name]/good` tag — and, for top-N and single-winner runs, any `variant-K` tags; in exploration the `variant-K` tags are the only references to the variants, so keep them until the user has said which to keep or discard — then offer the user: squash-merge into the `base` branch recorded in `config.yaml` (the one place the run commits to the user's checkout, and only when they say yes; a run that came from a 1.2 install also carries its `.gitignore` commit — drop it from the merge), push and open a PR, or leave the branch for review. Never merge without asking. Offer the merge only if the user's checkout is on `base.branch` with nothing staged or modified — otherwise (including a run that started from a detached commit) offer the PR or the branch, and never checkout, stash, or reset in their checkout to make a merge possible. Once they have decided, remove the worktree (`git -C <target repo> worktree remove <worktree>` — the branch keeps every commit), unless they want to inspect it first. If it refuses because of untracked or modified files, never add `--force`: show `git -C <worktree> status` and ask the user.

---

## examples

### example 1: optimizing a Next.js Lighthouse score

**Configuration:**
- Target files: `src/app/page.tsx`, `src/components/Hero.tsx`, `next.config.js`
- Output mode: single-winner / Rollback: git
- Worktree setup: `npm ci`
- Guards: `npm run build`, `npm test`, and a guard that (re)starts `npm start` from the worktree on port 3100 and polls until it answers (capped with `timeout 60`) — the user's own dev server on 3000 serves their checkout, not the mutation
- Objective: pass-count over the three command checks:
  - command: `npx lighthouse http://localhost:3100 --output=json --quiet` / extract: `.categories.performance.score` / check: `>= 0.9`
  - command: `npx lighthouse http://localhost:3100 --output=json --quiet` / extract: `.audits.largest-contentful-paint.numericValue` / check: `< 2500`
  - command: `du -sk .next | cut -f1` / extract: `raw` / check: `< 5120` (kilobytes — `du -sk` is portable; GNU-only `du -sb` is not)
- Constraint:
  - judgment: "Does the page display every section listed in `tasks/sections.md`, with its buttons, forms, and links responding?" (the section inventory is written at setup; a blind subagent fetches the page each run and answers against it)
- Timeout: 300s (build ~60s + 3 Lighthouse runs at ~60s each)
- Max iterations: 20
- Runs: 3

Note: The two Lighthouse evaluators share the same command string. Lighthouse runs once per run (3 total), not twice per run. Both evaluators parse the same output.

**Result:** Baseline 33% → Final 100% in 4 kept experiments. Key wins: dynamic imports for below-the-fold components, CSS modules replacing global styles, optimizeCss in next.config.js.

### example 2: optimizing a Python API response time

**Configuration:**
- Target files: `src/api/routes/search.py`, `src/api/db/queries.py`, `src/api/cache.py`
- Output mode: single-winner / Rollback: git
- Worktree setup: `uv sync`
- Guards: `pytest tests/`, `python -c "from src.api import app"`, and a guard that (re)starts the API from the worktree on port 8100 (`uvicorn src.api:app --port 8100` in the background, polled until it answers)
- Evaluators:
  - command: `hey -n 200 -c 10 'http://localhost:8100/api/search?q=test' | awk '/Average:/{print $2*1000}'` / extract: `raw` / check: `< 100` (avg latency, ms)
  - command: `hey -n 200 -c 10 'http://localhost:8100/api/search?q=test' | awk '/99% in/{print $3*1000}'` / extract: `raw` / check: `< 500` (p99 latency, ms)
  - command: `python -c "import tracemalloc; tracemalloc.start(); from src.api import app; print(tracemalloc.get_traced_memory()[1])"` / extract: `raw` / check: `< 52428800`
  - judgment: "For each query in `tasks/queries.md`, does the search endpoint return every result listed for it there?"
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
  - `docker compose -p autoresearch-[name] build --quiet`
  - `docker compose -p autoresearch-[name] up -d && timeout 60 sh -c 'until docker compose -p autoresearch-[name] exec api curl -sf http://localhost:8080/health; do sleep 1; done'` (the poll loop is itself capped — an uncapped `until` can hang the run; the project name keeps the run's containers apart from the user's — if their own stack also publishes 8080, map the run's to another host port in a compose override file kept in the artifacts directory)
  - `docker compose -p autoresearch-[name] exec api go test ./...`
- Evaluators:
  - command: `hey -n 500 -c 20 http://localhost:8080/api/orders | awk '/Average:/{print $2*1000}'` / extract: `raw` / check: `< 50`
  - command: `hey -n 500 -c 20 http://localhost:8080/api/orders | awk '/99% in/{print $3*1000}'` / extract: `raw` / check: `< 200`
  - command: `docker stats --no-stream --format '{{.MemUsage}}' $(docker compose -p autoresearch-[name] ps -q api) | awk -F'MiB' 'NF>1{print $1}'` / extract: `raw` / check: `< 256` (if docker reports GiB the extraction yields nothing and the eval fails — correct, since GiB-scale usage exceeds the threshold anyway)
  - judgment: "Does the /api/orders response contain every field listed in `tasks/order-schema.json`, each with a value of the listed type?"
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
- Objective (metric): `hyperfine --warmup 3 --export-json /tmp/hf.json './process testdata/large.csv' && jq -r '.results[0].mean' /tmp/hf.json` / extract: `raw` / direction: lower / target: `0.5` seconds (hyperfine has no stdout JSON mode — export to a file and read it back)
- Constraints:
  - command: `diff -q <(./process testdata/large.csv) testdata/large.golden` / check: `exit 0` (output identical to the saved baseline; a missing golden file makes `diff` exit non-zero, so it fails loudly — a `diff … | wc -l` form would pass on a missing file)
  - command: `/usr/bin/time -l ./process testdata/large.csv 2>&1 | awk '/maximum resident/{print $1}'` / extract: `raw` / check: `< 104857600` (macOS; on Linux use `/usr/bin/time -v`, the `Maximum resident set size (kbytes)` line, and a threshold of `102400`)
  - judgment: "Does the progress output show a percentage, the current row count, and an ETA on every update line?"
- Timeout: 300s
- Max iterations: 20
- Runs: 5

Note: `hyperfine` runs the command multiple times internally and reports mean execution time in seconds. Higher runs (5) help smooth out variance; the noise margin comes from the spread of three baseline evaluations, and every would-be keep is re-timed alternating with the anchor.

**Result:** Baseline mean 1.8s → 0.41s, under the 0.5s target, in 6 kept experiments. Key wins: replaced encoding/json with jsoniter for parsing, added worker pool for concurrent row processing, switched from map to slice for ordered results, reduced allocations by reusing buffers in pipeline.

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
2. **Measured, not guessed** — binary checks or a raw metric objective, no scales, no vibes; thresholds calibrated near the baseline; judges calibrated on the user's examples; the keep margin taken from the measured baseline noise
3. **Changed one thing at a time** — so you know what helped
4. **Kept a complete log** — every experiment recorded in changelog
5. **Ran guards before evaluating** — broken code never reached scoring
6. **Used anchor-based rollback** — the per-run tag / latest `-good` dir / kept export id, advanced on every keep — never `HEAD~1`
7. **Ran isolated** — mutations happened in a worktree (or under the snapshot mechanism); the user's checkout, branch, and uncommitted work never changed
8. **Resumable** — every setting lived in `config.yaml`, not only in the conversation
9. **Judged blind** — no judgment eval was graded by the agent that authored the mutation
10. **Re-measured every keep** — no change was kept on the draw that selected it
11. **Kept the evals out of reach** — no mutation touched a test, golden file, task, or anything else outside the targets
12. **Improved the score** — measurable improvement from baseline to final (or, in exploration mode, delivered N genuinely distinct valid variants)
13. **Didn't overfit** — the target got better at the actual job, not just at passing evals
14. **Ran autonomously** — didn't stop to ask permission between experiments
15. **Re-read the changelog** — didn't repeat failed experiments or forget what worked

If the target "passes" all evals but actual quality hasn't improved — the evals are bad. Go back and write better evals.
