---
name: autoresearch
description: "Autonomously optimize any target — skill prompts or application code — by running an iterative mutation/evaluation loop. Define target files, evaluators (shell commands with thresholds and/or binary agent judgments), and guards. The agent mutates, measures, keeps or discards, and repeats. Based on Karpathy's autoresearch methodology. Use when: optimize this skill, improve this skill, run autoresearch on, make this skill better, self-improve skill, benchmark skill, eval my skill, run evals on, optimize this code, improve performance, optimize for lighthouse, reduce bundle size, speed up this endpoint, autoresearch this codebase."
---

# Autoresearch

Autonomously optimize any target — skill prompts, frontend performance, API latency, bundle size, or anything else you can measure. Define what to change, how to score it, and what must not break. The agent handles the rest.

This skill adapts Andrej Karpathy's autoresearch methodology to any optimization target. The core loop: mutate target files, run guards, evaluate against binary criteria, keep improvements, discard the rest.

---

## the core job

Take any set of target files, define what "good" looks like as binary pass/fail checks, then run an autonomous loop that:

1. Mutates the target files (one change per experiment)
2. Runs guard commands to verify nothing is broken
3. Scores the result against all evaluators
4. Keeps mutations that improve the score, discards the rest
5. Repeats until the score ceiling is hit, max iterations reached, or the user stops it

**Output:** Optimized target files on a dedicated `autoresearch/[name]` branch + `results.tsv` log + `changelog.md` of every mutation attempted + a live HTML dashboard.

---

## before starting: gather context

**STOP. Do not run any experiments until all fields below are confirmed with the user. Ask for any missing required fields before proceeding.**

Pick a short kebab-case run name `[name]` (e.g. `nextjs-perf`, `search-latency`) — it names the branch, the rollback tag, and the artifacts directory.

1. **Target files** — Explicit list of file paths the agent can edit. Nothing else is editable. Examples: `src/app/page.tsx, src/components/Hero.tsx, next.config.js` or `~/.claude/skills/my-skill/SKILL.md`

2. **Evaluators** — List of checks (3-6 recommended). Each one is either:

   **Command evaluator** — a shell command, extraction path, and threshold:
   ```
   command:  "npx lighthouse http://localhost:3000 --output=json --quiet"
   extract:  ".categories.performance.score"
   check:    ">= 0.9"
   ```
   Extraction formats: jq-style path for JSON (`.field.subfield`), `"field N"` for whitespace-delimited output, `"raw"` for plain numeric output.

   **Calibrate thresholds against the current baseline.** Binary scoring is blind to sub-threshold movement: if the baseline is 0.62 and the threshold is `>= 0.9`, an improvement to 0.85 scores exactly the same as no improvement and gets discarded. Either set thresholds just beyond the current value so progress can register, or use stepped thresholds — the same command as three evaluators with `>= 0.7`, `>= 0.8`, `>= 0.9` — so each increment flips an eval. See [references/eval-guide.md](references/eval-guide.md).

   **Judgment evaluator** — a binary yes/no question. Two things must be pinned down at setup:
   - **What is judged and how it is produced fresh each run.** For code: fetch the page, run the binary, read the build artifact. For skill targets: write a fixed set of 3-5 test prompts into `autoresearch-[name]/tasks/` at setup; each run executes the target skill against them in a fresh subagent. A judgment with no fresh artifact to inspect is ungrounded — the score would just measure the agent's optimism about its own edit.
   - **Who judges: a fresh subagent, blind to the experiment.** Dispatch a subagent that receives ONLY the artifact and the yes/no question — never the diff, the hypothesis, or the changelog. The agent that authored a mutation must not grade it; self-graded judgments say "yes" almost every time.

   Both types can be mixed in a single run. Prefer command evaluators wherever the quality is mechanically checkable (a deterministic "is the output identical" check belongs in a `diff`-based command eval, not a judgment).

   **Command deduplication:** When multiple evaluators share the same command string, the command runs once per run. All evaluators sharing that command parse the same output.

3. **Guards** — Shell commands that must exit 0 after every mutation (at least one required). If any guard fails, the mutation is auto-discarded without running evaluators.
   - Code: `npm run build`, `npm test`, `pytest tests/`
   - Skills: `echo ok` (trivial guard)

4. **Timeout** — Max seconds per experiment (required). The budget covers one full experiment: all guard commands plus all N evaluation runs combined. If exceeded, the experiment is auto-discarded. See "enforcing the timeout" below. Rule of thumb: `guard time + runs × (sum of unique evaluator command times) + 20% margin`.
   - Lighthouse, 3 runs: ~300s
   - API benchmarking, 3 runs: ~240s
   - Docker rebuild + integration tests: ~600s
   - Skill optimization, 5 LLM runs: ~600s

5. **Max iterations** — Max experiment cycles before stopping (required). Forces you to choose a compute budget. Can be set high (100) but must be explicit.

6. **Runs per experiment** — How many times to evaluate per mutation. Defaults to 5 if unspecified. 3 is fine for deterministic benchmarks. 5 for nondeterministic outputs (skill prompts, LLM judgments).

### scoring

Everything is binary — pass or fail. Command evaluators extract a value and check it against the threshold. Judgment evaluators are yes/no.

**Total score** = passes across all evaluators × all runs.
**Max score** = number of evaluators × runs per experiment.
**Pass rate** = score / max_score.

Each "run" executes all unique commands once (deduplicated), then scores all evaluators against the output. So 4 evaluators using 2 unique commands with 3 runs = 6 command executions total, 12 pass/fail scores. Max score = 12.

**Failed commands:** if a command exits non-zero, is killed by the timeout, or its extraction yields no numeric value, every evaluator bound to that command scores fail for that run. Never substitute a stale or guessed value.

**Experiments rejected before scoring** (guard failure or timeout) are never evaluated: log them with empty score/max_score/pass_rate fields in `results.tsv` and `null` pass_rate in the dashboard data. Do not fabricate a score for an experiment that was never measured.

---

## setup

Before the loop starts, run these steps in order:

1. **Collect configuration** from user — fields 1-5 are required; runs per experiment defaults to 5.
2. **Safety-review guards and evaluators.** These commands run unattended, dozens of times. Refuse or explicitly confirm anything destructive or outward-facing: deletes outside the artifacts dir, `sudo`, piping downloads to a shell, benchmarks pointed at production URLs.
3. **Resolve the target repo.** For each target file, run `git -C <dir-of-target> rev-parse --show-toplevel`. ALL target files must resolve to the SAME repository — call it the target repo, and run every git command below as `git -C <target repo>`. If targets span two repos, stop and ask the user to split the run. If a target is in no repo at all, see "non-git targets" below.
4. **Check git state** — run `git -C <target repo> status`. If uncommitted changes exist, abort: "Uncommitted changes detected. Please commit or stash before running autoresearch." Do NOT auto-commit user's work. Abort on detached HEAD as well — the run needs a branch to return to.
5. **Collision check.** If the tag `autoresearch/[name]/good`, the branch `autoresearch/[name]`, or the directory `autoresearch-[name]/` already exists, a previous run by that name exists. Ask the user: resume it (see recovery) or pick a new name. Never overwrite a previous run's `baselines/`.
6. **Read and understand all target files.** For code: understand the architecture, dependencies, and what each file does. For skills: read the full SKILL.md and any referenced files.
7. **Verify guards pass** on the current state, each prefixed with `timeout`. If any fails, stop and tell the user — the project is already broken.
8. **Create the run branch:** `git -C <target repo> checkout -b autoresearch/[name]`. Every experiment commit lands here; the user's branch is never touched.
9. **Create the working directory** `autoresearch-[name]/` at the target repo root. Add `autoresearch-*/` to `.gitignore` if not already present — and if that changed `.gitignore`, commit that single change now (`autoresearch: ignore artifacts directory`). The ignore entry must be committed before the first experiment, or a later discard's reset will unignore the artifacts and a subsequent commit-and-discard cycle will delete them from disk.
10. **Back up all target files** to `autoresearch-[name]/baselines/`, mirroring their repo-relative paths (e.g. `baselines/src/app/page.tsx`). These are the pre-run originals — they are an abort escape hatch, not a rollback mechanism.
11. **Tag the starting point:** `git -C <target repo> tag -f autoresearch/[name]/good`.
12. **Create artifacts:** `results.tsv` (with header row), `dashboard.html` (copy from [references/dashboard-template.html](references/dashboard-template.html), replace `__DATA_PLACEHOLDER__` with initial data JSON), and for skill targets the fixed test-task set under `autoresearch-[name]/tasks/`. Open the dashboard: `open autoresearch-[name]/dashboard.html` (macOS) / `xdg-open` (Linux).
13. **Run baseline** (experiment 0) — evaluate the current state without changing anything. Score all evaluators × all runs. Then:
    - 100% → inform user all evals already pass, stop.
    - max_score − 1 → warn the user the run is one pass from the ceiling and ask whether it's worth proceeding.
    - Otherwise → report the baseline score and proceed. Do not wait for an acknowledgment — the user may already be away.

---

## the experiment loop

Once the baseline is reported, the loop runs autonomously. Do not pause to ask the user between experiments.

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

Bad mutations:
- Rewriting everything from scratch
- Changing 5 things at once
- Adding complexity without a specific hypothesis
- Vague changes ("make it better")

**3. Verify the anchor** — Confirm `autoresearch/[name]/good` points at HEAD (`git -C <target repo> rev-parse` both). It always should: KEEP advances it and DISCARD resets to it. If they diverge, something went wrong — stop and run recovery instead of force-tagging over the discrepancy.

**4. Mutate** — Edit target file(s). Stage ONLY the declared target files (`git -C <target repo> add <each target path>` — never `git add -A`; sweeping in artifacts or unrelated files is how a later discard deletes them). Commit as a single commit with `--no-verify` (a pre-commit hook that rewrites files would silently change the mutation; one that rejects the message format would derail the loop). Message format: `autoresearch: [short description of change]`. Verify the target files are clean in `git status` after committing.

**5. Guard** — Run all guard commands, each prefixed with `timeout <remaining>s` (see "enforcing the timeout").
- Any guard fails → `git -C <target repo> reset --hard autoresearch/[name]/good`, log as `guard_fail`, go to step 0.
- Budget exceeded or a command killed (exit 124) → reset the same way, log as `timeout`, go to step 0.

**6. Evaluate** — Run all evaluators × N runs, every command timeout-prefixed, tracking the remaining experiment budget. For each run: execute all unique commands once (deduplicated), then score every evaluator against the output. For judgment evaluators: produce the fresh artifact, then dispatch a blind subagent with only the artifact and the question. If the budget runs out mid-evaluation, treat the experiment as `timeout` (reset, log, go to step 0).

**7. Decide:**
- Score improved — and, if any evaluator is a judgment or otherwise non-deterministic, improved by **at least 2 passes** → **KEEP.** A +1 blip on a noisy eval is indistinguishable from variance; treating it as a win ratchets noise into the baseline. (With only deterministic command evals, any strict improvement counts.)
- Score equal, the change strictly simplifies the target files (net-negative diff), and all guards passed → **KEEP** as a simplification win. Note it as such in the description.
- Otherwise (equal without simplification, worse, or +1 within noise) → **DISCARD.** `git -C <target repo> reset --hard autoresearch/[name]/good`.
- **On KEEP, immediately advance the anchor:** `git -C <target repo> tag -f autoresearch/[name]/good`. The tag must always point at the current baseline — if the run stops or crashes right after a keep, an un-advanced tag makes recovery silently destroy the best result.

**8. Log** — Append to `results.tsv`. Update `dashboard.html` with new inline data. Append to `changelog.md` (see format below).

**9. Check stop conditions:**
- User manually stops → stop, deliver results.
- Max iterations reached → stop, deliver results.
- **Ceiling reached:** the standing baseline score is ≥ max_score − 1 AND the last 3 consecutive experiments (kept or discarded) failed to improve it → stop, deliver results. Judge this against the baseline's own score — never against the logged score of a discarded mutation, which is lower by definition.
- (Per-experiment timeout is handled in steps 5-6 — that experiment is discarded, loop continues.)
- Otherwise → go to step 0.

**NEVER STOP between experiments to ask the user.** They may be away. Run autonomously until a stop condition is met or the user interrupts.

**If you run out of ideas:** Re-read the failing outputs. Re-read the changelog. Try combining two previous near-miss mutations. Try a completely different approach to the same problem. Try removing things instead of adding them — a simplification that holds the score is a keepable win.

### enforcing the timeout

An agent cannot interrupt a hung shell command, so the timeout only exists if every command is wrapped. Record the experiment start time. Before each guard or evaluator command, compute the remaining budget and prefix the command with `timeout <remaining>s` (GNU coreutils; on macOS `brew install coreutils` provides it as `timeout` or `gtimeout`). Exit code 124 means the command was killed — treat it as the budget being exceeded. Never run a guard or evaluator bare: a mutation can pass every guard and still hang at runtime (an accidental infinite loop passes `py_compile` and unit tests that don't call the hot path), and an unwrapped evaluator would block the loop forever.

---

## git strategy

- **Dedicated branch.** All experiment commits land on `autoresearch/[name]`, created at setup. The user's branch never sees a mutation commit, and `reset --hard` never runs against their live branch.
- **One commit per experiment, target files only.** Stage exactly the declared target files. Never `git add -A` / `git add .`.
- **Tag-based rollback.** The per-run tag `autoresearch/[name]/good` always points at the current baseline: set at setup, advanced on every KEEP, reset target on every discard/guard_fail/timeout. More robust than `HEAD~1`, and namespaced so concurrent or repeated runs can't clobber each other.
- **Clean state required.** Checked at setup. Uncommitted changes → abort.
- **Every git command runs against the target repo** (`git -C <target repo> …`) — the repo containing the target files, which is not necessarily the cwd's repo.
- Commit message format: `autoresearch: [short description]`

### non-git targets

If a target file lives outside any git repo (common for skill targets like `~/.claude/skills/...`):
1. Offer to `git init` the target's directory and commit the current state — then the normal git strategy applies, with that new repo as the target repo.
2. If the user declines, use copy-based rollback: keep a `good/` mirror of all target files inside `autoresearch-[name]/` (alongside `baselines/`). KEEP = refresh the mirror from the targets; DISCARD/guard_fail/timeout = copy the mirror back over the targets. The mechanics of the loop are unchanged — the mirror plays the role of the tag.

Never mix mechanisms in one run: if rollback is git, every target must be in the target repo; a `reset --hard` that reverts only some targets leaves "discarded" mutations silently alive in the others.

### recovery

If the session ends mid-experiment (crash, disconnect, context limit):
1. Check whether target files differ from the last committed state (`git -C <target repo> diff`).
2. If dirty: `git -C <target repo> reset --hard autoresearch/[name]/good`. Because the tag advances on every KEEP, this restores the latest kept state and never loses kept work.
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

All artifacts live in `autoresearch-[name]/` at the target repo root:

```
autoresearch-[name]/
├── dashboard.html          # live browser dashboard (auto-refreshes, data inlined)
├── results.tsv             # score log for every experiment
├── changelog.md            # mutation log with reasoning (WHY things worked/failed)
├── tasks/                  # fixed test prompts for judgment evals (skill targets)
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
  "current_experiment": 7,
  "max_iterations": 30,
  "baseline_score": 33.0,         // pass-rate PERCENT (0-100) of experiment 0 — not a raw count
  "best_score": 83.0,             // highest pass-rate PERCENT among baseline + kept experiments
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

`eval_breakdown` reflects the most recent full evaluation of the current baseline — not a cumulative tally across all experiments. When the loop stops, set `status` to `"complete"` (or `"error"` if the run was abandoned) so the dashboard shows a finished state.

### results.tsv

Tab-separated with columns: experiment, score, max_score, pass_rate, status, description.

Status values: `baseline`, `keep`, `discard`, `guard_fail`, `timeout`. Guard failures and timeouts were never scored — leave their score, max_score, and pass_rate fields empty.

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

When the loop stops (max iterations, ceiling hit, or user interrupts), present:

1. **Score summary:** Baseline score → Final score (% improvement)
2. **Total experiments run:** kept / discarded / guard failures / timeouts
3. **Top 3 most impactful changes** (from the changelog)
4. **Remaining failure patterns** (what still fails, if anything)
5. **Location of all artifacts**
6. **Integration:** the optimized state lives on the `autoresearch/[name]` branch. Delete the run tag (`git -C <target repo> tag -d autoresearch/[name]/good`), then offer the user: squash-merge into their original branch, push and open a PR, or leave the branch for review. Never merge without asking.

---

## examples

### example 1: optimizing a Next.js Lighthouse score

**Configuration:**
- Target files: `src/app/page.tsx`, `src/components/Hero.tsx`, `next.config.js`
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
- Rollback: `~/.claude/skills` is usually not a git repo — offer `git init` on it, else copy-based rollback (see "non-git targets")
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

---

## the test

A good autoresearch run:

1. **Started with a baseline** — never changed anything before measuring
2. **Used binary evals only** — no scales, no vibes — with thresholds calibrated near the baseline
3. **Changed one thing at a time** — so you know what helped
4. **Kept a complete log** — every experiment recorded in changelog
5. **Ran guards before evaluating** — broken code never reached scoring
6. **Used tag-based rollback** — the per-run `autoresearch/[name]/good` tag, advanced on every keep — not `HEAD~1`
7. **Ran on a dedicated branch** — the user's branch never saw a mutation commit
8. **Judged blind** — no judgment eval was graded by the agent that authored the mutation
9. **Improved the score** — measurable improvement from baseline to final
10. **Didn't overfit** — the target got better at the actual job, not just at passing evals
11. **Ran autonomously** — didn't stop to ask permission between experiments
12. **Re-read the changelog** — didn't repeat failed experiments or forget what worked

If the target "passes" all evals but actual quality hasn't improved — the evals are bad. Go back and write better evals.
