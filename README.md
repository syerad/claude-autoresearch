# Autoresearch

Autonomous optimization for Claude Code skills and codebases.

## What it does

Define target files, evaluators (shell commands with thresholds and/or binary agent judgments), and guards. The agent runs an iterative mutation/evaluation loop: mutate one thing, check guards, score against evaluators, keep or discard, repeat. Works on anything measurable — skill prompts, Lighthouse scores, API latency, bundle size, Docker container memory, CLI execution time. Based on [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) methodology.

## Installation

### From the plugin marketplace

```
/plugin install autoresearch@claude-plugins-official
```

### From GitHub

```
claude plugins marketplace add syerad/autoresearch
claude plugins install autoresearch@autoresearch
```

## Quick start

Invoke with `/autoresearch` or say "run autoresearch on this codebase."

The skill asks for 6 fields before starting:

1. **Target files** — which files the agent can edit
2. **Evaluators** — command checks (shell command + threshold) and/or judgment checks (binary yes/no questions)
3. **Guards** — commands that must pass after every mutation (at least one required)
4. **Timeout** — max seconds per experiment
5. **Max iterations** — experiment budget
6. **Runs per experiment** — evaluations per mutation (default: 5)

## How it works

1. **Baseline** — measure current state before changing anything
2. **Mutate** — make ONE targeted change to the target files
3. **Guard** — verify nothing is broken (build passes, tests pass, site responds)
4. **Evaluate** — score against all evaluators. Everything is binary — pass or fail
5. **Decide** — score improved? Keep. Same or worse? Discard and git reset
6. **Repeat** — autonomous loop until max iterations or 95%+ pass rate

All changes are git-committed. Failed experiments are rolled back with tag-based reset. A live HTML dashboard tracks progress.

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

## License

MIT
