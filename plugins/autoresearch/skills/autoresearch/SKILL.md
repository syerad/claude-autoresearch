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

**Output:** Optimized target files + `results.tsv` log + `changelog.md` of every mutation attempted + a live HTML dashboard.

---

## before starting: gather context

**STOP. Do not run any experiments until all fields below are confirmed with the user. Ask for any missing fields before proceeding.**

1. **Target files** — Explicit list of file paths the agent can edit. Nothing else is editable. Examples: `src/app/page.tsx, src/components/Hero.tsx, next.config.js` or `~/.claude/skills/my-skill/SKILL.md`

2. **Evaluators** — List of checks (3-6 recommended). Each one is either:

   **Command evaluator** — a shell command, extraction path, and threshold:
   ```
   command:  "npx lighthouse http://localhost:3000 --output=json"
   extract:  ".categories.performance.score"
   check:    ">= 0.9"
   ```
   Extraction formats: jq-style path for JSON (`.field.subfield`), `"field N"` for whitespace-delimited output, `"raw"` for plain numeric output.

   **Judgment evaluator** — a binary yes/no question the agent answers:
   ```
   judgment: "Does the page still display all original content sections and interactive elements?"
   ```
   For code optimization, the agent uses available tools to answer (fetch page, read command output, inspect build artifacts). If the judgment requires visual inspection, say so explicitly.

   Both types can be mixed in a single run. See [references/eval-guide.md](references/eval-guide.md) for how to write good evals.

   **Command deduplication:** When multiple evaluators share the same command string, the command runs once per run. All evaluators sharing that command parse the same output.

3. **Guards** — Shell commands that must exit 0 after every mutation (at least one required). If any guard fails, the mutation is auto-discarded without running evaluators.
   - Code: `npm run build`, `npm test`, `pytest tests/`
   - Skills: `echo ok` (trivial guard)

4. **Timeout** — Max seconds per experiment (required). If exceeded, the experiment is auto-discarded.
   - Lighthouse runs: ~120s
   - API benchmarking: ~180s
   - Large builds + integration tests: ~300-600s
   - Skill optimization: ~60s

5. **Max iterations** — Max experiment cycles before stopping (required). Forces you to choose a compute budget. Can be set high (100) but must be explicit.

6. **Runs per experiment** — How many times to evaluate per mutation. Default: 5. More runs = more reliable scores but slower. 3 is fine for deterministic benchmarks. 5 for nondeterministic outputs (skill prompts, LLM judgments).

### scoring

Everything is binary — pass or fail. Command evaluators extract a value and check it against the threshold. Judgment evaluators are yes/no.

**Total score** = passes across all evaluators × all runs.
**Max score** = number of evaluators × runs per experiment.
**Pass rate** = score / max_score.

Each "run" executes all unique commands once (deduplicated), then scores all evaluators against the output. So 4 evaluators using 2 unique commands with 3 runs = 6 command executions total, 12 pass/fail scores. Max score = 12.

---

## setup

Before the loop starts, run these steps in order:

1. **Collect configuration** from user. All 6 fields required — ask for any missing.
2. **Check git state** — run `git status`. If uncommitted changes exist, abort: "Uncommitted changes detected. Please commit or stash before running autoresearch." Do NOT auto-commit user's work.
3. **Read and understand all target files.** For code: understand the architecture, dependencies, and what each file does. For skills: read the full SKILL.md and any referenced files.
4. **Verify guards pass** on the current state. Run every guard command. If any fails, stop and tell the user — the project is already broken.
5. **Back up all target files** to `autoresearch-[name]/baselines/` directory. Mirror full relative paths from the project root to avoid name collisions (e.g., `baselines/src/app/page.tsx`).
6. **Git commit** current state as clean starting point. Tag as `autoresearch/good`.
7. **Create working directory** `autoresearch-[name]/` in the project root. Add `autoresearch-*/` to `.gitignore` if not already present.
8. **Create artifacts:** `results.tsv` (with header row) and `dashboard.html` (copy from [references/dashboard-template.html](references/dashboard-template.html), replace `__DATA_PLACEHOLDER__` with initial data JSON).
9. **Open dashboard** in browser: `open autoresearch-[name]/dashboard.html` (macOS).
10. **Run baseline** (experiment 0) — evaluate the current state without changing anything. Score all evaluators × all runs.
11. **Check baseline score:**
    - 100% → inform user all evals already pass, stop.
    - 95%+ → warn user and ask whether to proceed.
    - Otherwise → confirm score with user and proceed.

---

## the experiment loop

Once baseline is confirmed, the loop runs autonomously. Do not pause to ask the user between experiments.

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
- Simplify — if score holds, simpler is better

Bad mutations:
- Rewriting everything from scratch
- Changing 5 things at once
- Adding complexity without a specific hypothesis
- Vague changes ("make it better")

**3. Tag** — `git tag -f autoresearch/good` on current HEAD.

**4. Mutate** — Edit target file(s). Stage ALL changes. Git commit as a single commit. Message format: `autoresearch: [short description of change]`

**5. Guard** — Run all guard commands within the timeout.
- Any guard fails → `git reset --hard autoresearch/good`, log as `guard_fail`, go to step 0.
- Timeout exceeded → `git reset --hard autoresearch/good`, log as `timeout`, go to step 0.

**6. Evaluate** — Run all evaluators × N runs. For each run: execute all unique commands once (deduplicated), then score every evaluator against the output. For judgment evaluators: inspect the relevant output and answer the yes/no question.

**7. Decide:**
- Score improved → **KEEP.** This commit is the new baseline.
- Score same or worse → **DISCARD.** `git reset --hard autoresearch/good`.

**8. Log** — Append to `results.tsv`. Update `dashboard.html` with new inline data. Append to `changelog.md` (see format below).

**9. Check stop conditions:**
- User manually stops → stop, deliver results.
- Max iterations reached → stop, deliver results.
- 95%+ pass rate for 3 consecutive experiments (kept or discarded) → stop, deliver results.
- (Per-experiment timeout is handled in step 5 — that experiment is discarded, loop continues.)
- Otherwise → go to step 0.

**NEVER STOP between experiments to ask the user.** They may be away. Run autonomously until a stop condition is met or the user interrupts.

**If you run out of ideas:** Re-read the failing outputs. Re-read the changelog. Try combining two previous near-miss mutations. Try a completely different approach to the same problem. Try removing things instead of adding them. Simplification that maintains the score is a win.

---

## git strategy

- **One commit per experiment.** All target file changes must be in a single commit. Never split across multiple commits.
- **Tag-based rollback.** Before each experiment, tag HEAD as `autoresearch/good`. To discard: `git reset --hard autoresearch/good`. This is more robust than `HEAD~1`.
- **Clean state required.** Checked at setup. Uncommitted changes → abort.
- Commit message format: `autoresearch: [short description]`

### recovery

If the session ends mid-experiment (crash, disconnect, context limit):
1. Check if target files differ from last committed state (`git diff`).
2. If dirty: offer to reset to `autoresearch/good` tag or restore from `baselines/`.
3. Re-read `results.tsv` and `changelog.md` to resume the loop.

---

## changelog format

After every experiment (kept or discarded), append to `changelog.md`:

```markdown
## Experiment [N] — [keep/discard/guard_fail/timeout]

**Score:** [X]/[max] ([percent]%)
**Change:** [One sentence describing what was changed]
**Reasoning:** [Why this change was expected to help]
**Result:** [What actually happened — which evals improved/declined]
**Failing outputs:** [Brief description of what still fails, if anything]
```

This changelog is the most valuable artifact. It's a research log that persists WHY things worked or failed. The agent re-reads the last 10 entries at the start of each loop iteration.

---

## artifacts

All artifacts live in `autoresearch-[name]/` in the project root:

```
autoresearch-[name]/
├── dashboard.html          # live browser dashboard (auto-refreshes, data inlined)
├── results.tsv             # score log for every experiment
├── changelog.md            # mutation log with reasoning (WHY things worked/failed)
└── baselines/              # original target files before any changes
    └── [mirrored paths]    # full relative paths from project root
```

### dashboard

Copy from [references/dashboard-template.html](references/dashboard-template.html). Replace `__DATA_PLACEHOLDER__` with the JSON data object. Rewrite the entire `dashboard.html` file with updated inline data after each experiment (avoids `file://` CORS issues with `fetch()`). The template includes `<meta http-equiv="refresh" content="10">` for auto-refresh.

Dashboard data structure (embedded as `<script>const DATA = {...}</script>` in the HTML):

```json
{
  "name": "autoresearch-nextjs-perf",
  "status": "running",
  "current_experiment": 7,
  "max_iterations": 30,
  "timeout_seconds": 120,
  "baseline_score": 33.0,
  "best_score": 83.0,
  "experiments": [
    {
      "id": 0,
      "score": 4,
      "max_score": 12,
      "pass_rate": 33.0,
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

When the loop stops, set `status` to `"complete"` so the dashboard shows a finished state.

### results.tsv

Tab-separated with columns: experiment, score, max_score, pass_rate, status, description.

Status values: `baseline`, `keep`, `discard`, `guard_fail`, `timeout`.

```
experiment	score	max_score	pass_rate	status	description
0	4	12	33.0%	baseline	original code — no changes
1	6	12	50.0%	keep	dynamic import for Hero component
2	6	12	50.0%	discard	added priority to hero image — no change
3	4	12	33.0%	guard_fail	aggressive tree shaking — build broke
4	4	12	33.0%	timeout	full image optimization pipeline — exceeded 120s
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

---

## examples

### example 1: optimizing a Next.js Lighthouse score

**Configuration:**
- Target files: `src/app/page.tsx`, `src/components/Hero.tsx`, `next.config.js`
- Guards: `npm run build`, `npm test`
- Evaluators:
  - command: `npx lighthouse http://localhost:3000 --output=json` / extract: `.categories.performance.score` / check: `>= 0.9`
  - command: `npx lighthouse http://localhost:3000 --output=json` / extract: `.audits.largest-contentful-paint.numericValue` / check: `< 2500`
  - command: `du -sb .next | cut -f1` / extract: `raw` / check: `< 5242880`
  - judgment: "Does the page still display all original content sections and interactive elements?"
- Timeout: 120s
- Max iterations: 20
- Runs: 3

Note: The two Lighthouse evaluators share the same command. Lighthouse runs once per run (3 total), not twice per run. Both evaluators parse the same output.

**Result:** Baseline 33% → Final 100% in 4 kept experiments. Key wins: dynamic imports for below-the-fold components, CSS modules replacing global styles, optimizeCss in next.config.js.

### example 2: optimizing a Python API response time

**Configuration:**
- Target files: `src/api/routes/search.py`, `src/api/db/queries.py`, `src/api/cache.py`
- Guards: `pytest tests/`, `python -c "from src.api import app"`
- Evaluators:
  - command: `hey -n 200 -c 10 http://localhost:8000/api/search?q=test -o json` / extract: `.avg_latency_ms` / check: `< 100`
  - command: `hey -n 200 -c 10 http://localhost:8000/api/search?q=test -o json` / extract: `.p99_latency_ms` / check: `< 500`
  - command: `python -c "import tracemalloc; tracemalloc.start(); from src.api import app; print(tracemalloc.get_traced_memory()[1])"` / extract: `raw` / check: `< 52428800`
  - judgment: "Does the search endpoint still return correct, complete results for a variety of query terms?"
- Timeout: 180s
- Max iterations: 30
- Runs: 3

Note: The two `hey` evaluators share the same command. `hey` runs once per run (3 total), both evaluators parse the same JSON output.

**Result:** Baseline 33% → Final 100% in 6 experiments. Key wins: Redis caching layer, explicit column SELECTs instead of SELECT *, PostgreSQL full-text search replacing LIKE queries, async parallelization of independent DB calls.

### example 3: optimizing a skill prompt

**Configuration:**
- Target files: `~/.claude/skills/diagram-generator/SKILL.md`
- Guards: `echo ok`
- Evaluators:
  - judgment: "Is all text legible with no truncated or overlapping words?"
  - judgment: "Uses only pastel/soft colors — no neon, bright red, or high-saturation?"
  - judgment: "Linear layout — left-to-right or top-to-bottom with no scattered elements?"
  - judgment: "Free of numbered steps, ordinals, or sequential numbering?"
- Timeout: 60s
- Max iterations: 15
- Runs: 5

**Result:** Baseline 80% → Final 97.5% in 5 experiments. Key wins: specific hex codes replacing vague "pastel colors" instruction, explicit anti-numbering rule, worked example showing correct diagram format.

### example 4: optimizing a Dockerized microservice

**Configuration:**
- Target files: `src/handlers/order.go`, `src/db/queries.go`, `docker-compose.yml`
- Guards:
  - `docker compose build --quiet`
  - `docker compose up -d && until docker compose exec api curl -sf http://localhost:8080/health; do sleep 1; done`
  - `docker compose exec api go test ./...`
- Evaluators:
  - command: `hey -n 500 -c 20 http://localhost:8080/api/orders -o json` / extract: `.avg_latency_ms` / check: `< 50`
  - command: `hey -n 500 -c 20 http://localhost:8080/api/orders -o json` / extract: `.p99_latency_ms` / check: `< 200`
  - command: `docker stats --no-stream --format '{{.MemUsage}}' api | grep -oP '\d+\.?\d*(?=MiB)'` / extract: `raw` / check: `< 256`
  - judgment: "Does the /api/orders endpoint still return correct, complete order data matching the original response schema?"
- Timeout: 300s
- Max iterations: 25
- Runs: 3

Note: Timeout is higher (300s) because `docker compose build` adds rebuild time per experiment. The guard includes a health check polling loop so evaluators don't hit a container that's still starting. The two `hey` evaluators share a single command per run.

**Result:** Baseline 25% → Final 100% in 5 experiments. Key wins: connection pooling in database layer, query batching for N+1 problem, response payload trimming with field selection, enabling gzip compression in Docker reverse proxy config.

### example 5: optimizing a Go CLI tool's execution speed

**Configuration:**
- Target files: `cmd/process/main.go`, `internal/parser/parser.go`, `internal/pipeline/pipeline.go`
- Guards:
  - `go build ./cmd/process`
  - `go test ./...`
- Evaluators:
  - command: `hyperfine --warmup 3 --json './process testdata/large.csv'` / extract: `.results[0].mean` / check: `< 0.5`
  - command: `go build -o /dev/null ./cmd/process 2>&1 | wc -l` / extract: `raw` / check: `< 5`
  - command: `/usr/bin/time -l ./process testdata/large.csv 2>&1 | grep 'maximum resident' | awk '{print $1}'` / extract: `raw` / check: `< 104857600`
  - judgment: "Does the CLI tool produce identical output for testdata/large.csv compared to the baseline output?"
- Timeout: 120s
- Max iterations: 20
- Runs: 5

Note: `hyperfine` runs the command multiple times internally and reports mean execution time in seconds. Higher runs (5) help smooth out variance. The judgment eval compares against a saved baseline output to catch correctness regressions.

**Result:** Baseline 40% → Final 100% in 6 experiments. Key wins: replaced encoding/json with jsoniter for parsing, added worker pool for concurrent row processing, switched from map to slice for ordered results, reduced allocations by reusing buffers in pipeline.

---

## the test

A good autoresearch run:

1. **Started with a baseline** — never changed anything before measuring
2. **Used binary evals only** — no scales, no vibes
3. **Changed one thing at a time** — so you know what helped
4. **Kept a complete log** — every experiment recorded in changelog
5. **Ran guards before evaluating** — broken code never reaches scoring
6. **Used tag-based rollback** — `autoresearch/good`, not `HEAD~1`
7. **Improved the score** — measurable improvement from baseline to final
8. **Didn't overfit** — the target got better at the actual job, not just at passing evals
9. **Ran autonomously** — didn't stop to ask permission between experiments
10. **Re-read the changelog** — didn't repeat failed experiments or forget what worked

If the target "passes" all evals but actual quality hasn't improved — the evals are bad. Go back and write better evals.
