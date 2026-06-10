# Autoresearch

Autonomous optimization for Claude Code skills and codebases.

## What it does

Define target files, evaluators (shell commands with thresholds and/or binary agent judgments), and guards. The agent runs an iterative mutation/evaluation loop: mutate one thing, check guards, score against evaluators, keep or discard, repeat. Works on anything measurable — skill prompts, Lighthouse scores, API latency, bundle size, Docker container memory, CLI execution time. Based on [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) methodology.

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

Plus an **output mode** — `single-winner` (one optimized result), `top-N` (finalists to pick from), or `exploration` (a portfolio of distinct valid variants, e.g. bull/base/bear forecasts) — and a **rollback mechanism** (git, snapshot directory, API snapshot, or manual-confirm for targets outside version control).

## How it works

1. **Baseline** — measure current state before changing anything
2. **Mutate** — make ONE targeted change to the target files
3. **Guard** — verify nothing is broken (build passes, tests pass, site responds)
4. **Evaluate** — score against all evaluators. Everything is binary — pass or fail
5. **Decide** — score improved? Keep. Same or worse? Discard and git reset
6. **Repeat** — autonomous loop until the user stops it, max iterations are reached, or the score sits at the ceiling for 3 consecutive experiments

All experiments run on a dedicated `autoresearch/[name]` branch — the user's branch is never touched. Failed experiments are rolled back via a per-run tag that advances on every kept improvement. A live HTML dashboard tracks progress.

## Examples

### Lighthouse SEO optimization

```
Target:      header.php, hooks.php
Evaluator:   npx lighthouse --only-categories=seo (score >= 0.95)
Guard:       curl -sf http://localhost:8080/
Timeout:     120s, Max iterations: 50, Runs: 3
```

Result: 85% → 100% in 6 experiments. Fixes: meta description fallback, descriptive link text replacing generic "here" and "Learn More".

### API response time

```
Target:      routes/search.py, db/queries.py, cache.py
Evaluators:  hey benchmark (avg < 100ms, p99 < 500ms), memory < 50MB
Guards:      pytest, import check
Timeout:     180s, Max iterations: 30, Runs: 3
```

Result: 33% → 100% in 6 experiments. Fixes: Redis caching, explicit column SELECTs, full-text search, async parallelization.

### Skill prompt optimization

```
Target:      ~/.claude/skills/diagram-generator/SKILL.md
Evaluators:  4 judgment evals (text legibility, colors, layout, no numbering)
Guard:       echo ok
Timeout:     60s, Max iterations: 15, Runs: 5
```

Result: 80% → 97.5% in 5 experiments. Fixes: specific hex codes for colors, anti-numbering rule, worked example.

## Configuration

### Command evaluators

```
command:  shell command to run
extract:  how to get the value (jq path for JSON, "field N" for delimited, "raw" for plain number)
check:    threshold (">= 0.9", "< 100", "< 5242880")
```

### Judgment evaluators

A binary yes/no question the agent answers by inspecting the output.

### Command deduplication

When multiple evaluators share the same command, it runs once per run. All evaluators parse the same output. For example, two Lighthouse evaluators (performance score and LCP) share a single Lighthouse invocation.

## Methodology

This skill adapts [Andrej Karpathy's autoresearch](https://github.com/karpathy/autoresearch) methodology to Claude Code. The original runs autonomous ML training experiments — mutate code, train for 5 minutes, check validation loss, keep or discard. We generalize this to any optimization target with measurable success criteria.

Built on the initial skill-only implementation by [olelehmann100kMRR/autoresearch-skill](https://github.com/olelehmann100kMRR/autoresearch-skill).

## License

MIT
