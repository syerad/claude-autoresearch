# Autoresearch

Tell Claude what "better" means for your files, and it experiments until it gets there — keeping changes that measurably help, automatically undoing the ones that don't.

## What it does

You point it at one or more files — code, a prompt, marketing copy, a financial model — and describe what "better" means as simple pass/fail checks, like "the page loads in under 2.5 seconds" or "the email ends with one clear ask". Claude then improves the files the way a scientist would: change one thing, test it, keep the change only if the checks score better, undo it if they don't, and write down what happened. You get the improved files plus a full log of every experiment, and your originals are always recoverable.

It works on anything measurable: skill prompts, Lighthouse scores, API latency, bundle size, Docker container memory, CLI execution time, copy, documents. Based on [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) methodology.

## Installation

### From GitHub

```
claude plugins marketplace add syerad/claude-autoresearch
claude plugins install autoresearch@autoresearch
```

Or inside Claude Code:

```
/plugin marketplace add syerad/claude-autoresearch
/plugin install autoresearch@autoresearch
```

## Quick start

Invoke with `/autoresearch` — the skill opens with a single question: *"What are you trying to improve?"*

From your answer it infers a full draft configuration (target files, evals grounded in your examples, guards, rollback mechanism, output mode) and only asks about what it couldn't infer. Engineers who want full control can say "I want to edit directly" and fill in the raw fields:

1. **Target files** — which files the agent can edit
2. **Evaluators** — command checks (shell command + threshold) and/or judgment checks (binary yes/no questions, graded blind by a fresh subagent)
3. **Guards** — commands that must pass after every mutation (at least one required)
4. **Timeout** — max seconds per experiment, covering guards plus all evaluation runs
5. **Max iterations** — experiment budget
6. **Runs per experiment** — evaluations per mutation (defaults to 5)

Plus an **output mode**, picked to match your goal:

- `single-winner` — one optimized result. Want this when there's a clear target to hit: "make it faster", "score above 90".
- `top-N` — 2–3 strong finalists, side by side, and you pick. Want this when taste matters and you asked for options: marketing copy, microcopy, email templates.
- `exploration` — a portfolio of distinct valid variants instead of one winner. Want this when "best" is the wrong question: bull/base/bear forecasts, pricing scenarios, strategy directions.

And a **rollback mechanism** — git for files in a repository, a snapshot directory for binaries or files outside version control, API snapshot for live systems with export/restore, or manual-confirm as a last resort. Targets that can't be undone at all (sent emails, payments) are refused.

## How it works

1. **Baseline** — measure current state before changing anything
2. **Mutate** — make ONE targeted change to the target files
3. **Guard** — verify nothing is broken (build passes, tests pass, site responds)
4. **Evaluate** — score against all evaluators. Everything is binary — pass or fail
5. **Decide** — better score? Keep. Equal score but strictly simpler? Keep. Anything else — including a one-point blip on noisy checks, which must improve by at least 2 passes — is discarded and rolled back
6. **Repeat** — autonomous loop until the user stops it, max iterations are reached, or the score is within one pass of perfect and 3 consecutive experiments fail to improve it

Git runs happen on a dedicated `autoresearch/[name]` branch — the user's branch is never touched (non-git targets are protected by their snapshot mechanism instead). Failed experiments are rolled back via a per-run tag that advances on every kept improvement. When the run finishes, the result stays on the run branch and autoresearch offers to squash-merge, open a PR, or leave it for review — it never merges into your branch without asking. A live HTML dashboard tracks progress.

## Examples

### Lighthouse SEO optimization

```
Target:      header.php, hooks.php
Evaluator:   npx lighthouse --only-categories=seo (score >= 0.95)
Guard:       curl -sf http://localhost:8080/
Timeout:     300s, Max iterations: 50, Runs: 3
```

Result: 67% → 100% in 6 experiments. Fixes: meta description fallback, descriptive link text replacing generic "here" and "Learn More".

### API response time

```
Target:      routes/search.py, db/queries.py, cache.py
Evaluators:  hey benchmark (avg < 100ms, p99 < 500ms), memory < 50MB
Guards:      pytest, import check
Timeout:     240s, Max iterations: 30, Runs: 3
```

Result: 33% → 100% in 6 experiments. Fixes: Redis caching, explicit column SELECTs, full-text search, async parallelization.

### Skill prompt optimization

```
Target:      ~/.claude/skills/diagram-generator/SKILL.md
Evaluators:  4 judgment evals (text legibility, colors, layout, no numbering)
Guard:       echo ok
Timeout:     600s, Max iterations: 15, Runs: 5
```

Result: 80% → 95% (19/20) in 5 experiments. Fixes: specific hex codes for colors, anti-numbering rule, worked example.

## Configuration

### Command evaluators

```
command:  shell command to run
extract:  how to get the value (jq path for JSON, "field N" for delimited, "raw" for plain number)
check:    threshold (">= 0.9", "< 100", "< 5242880")
```

### Judgment evaluators

A binary yes/no question graded by a fresh subagent that sees only the produced artifact and the question — never the diff, and never graded by the agent that made the change.

### Command deduplication

When multiple evaluators share the same command, it runs once per run. All evaluators parse the same output. For example, two Lighthouse evaluators (performance score and LCP) share a single Lighthouse invocation.

## Methodology

This skill adapts [Andrej Karpathy's autoresearch](https://github.com/karpathy/autoresearch) methodology to Claude Code. The original runs autonomous ML training experiments — mutate code, train for 5 minutes, check validation loss, keep or discard. We generalize this to any optimization target with measurable success criteria.

Built on the initial skill-only implementation by [olelehmann100kMRR/autoresearch-skill](https://github.com/olelehmann100kMRR/autoresearch-skill).

## License

MIT
