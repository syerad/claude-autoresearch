# Eval Patterns Catalog

Domain-tagged catalog of known-good eval shapes. The front-door pulls 6–10 candidates from matching tags during setup, filters by whether user-provided examples discriminate, and presents 3–6 finalists.

## How to read this file

Each pattern is a YAML block with fields:

- `id` — kebab-case unique identifier
- `tags` — one or more of the tags below
- `type` — `command` (shell + extract + check) or `judgment` (yes/no question)
- `question` — the eval itself (required for judgment; descriptive for command)
- `command_template` — (command type only) shell command with `{target}` placeholder
- `extract_template` — (command type only) extraction path
- `check_template` — (command type only) threshold comparison
- `applies_when` — one-line description of when this pattern is a candidate
- `example_pass` — a short example of output that passes
- `example_fail` — a short example of output that fails

## Tags

- `code/performance`, `code/correctness`, `code/size`, `code/cost`
- `writing/tone`, `writing/structure`, `writing/length`, `writing/specificity`
- `docs/completeness`, `docs/citation`, `docs/format`
- `forecast/consistency`, `forecast/defensibility`, `forecast/fit-to-actuals`
- `visual/legibility`, `visual/layout`, `visual/palette`
- `prompt/output-shape`, `prompt/anti-pattern`

## Writing patterns

### `writing/specificity`

```yaml
- id: writing-opening-specificity
  tags: [writing/specificity]
  type: judgment
  question: "Does the first sentence reference a specific time, place, person, sensory detail, or concrete number?"
  applies_when: "User is optimizing opening lines, cold emails, intros, hooks, or leads."
  example_pass: "At 2 a.m. last Tuesday, Stripe's API returned a 504 we'd never seen before."
  example_fail: "I hope this email finds you well."
```

```yaml
- id: writing-concrete-claim
  tags: [writing/specificity]
  type: judgment
  question: "Is every claim in the output backed by a specific number, date, name, or source — no vague qualifiers like 'many', 'often', 'leading'?"
  applies_when: "User is optimizing persuasive or analytical writing where claims need to land."
  example_pass: "We cut CAC by 52% in Q3 2025 after three concrete changes."
  example_fail: "Many leading companies are seeing dramatic improvements."
```

### `writing/structure`

```yaml
- id: writing-single-concrete-ask
  tags: [writing/structure]
  type: judgment
  question: "Does the output end with exactly one specific, concrete ask — not 'let me know' or 'thoughts?'"
  applies_when: "User is optimizing messages that need a response (emails, outreach, requests)."
  example_pass: "Worth a 20-minute call on Tuesday at 2 pm ET?"
  example_fail: "Let me know if you're interested!"
```

```yaml
- id: writing-lede-first
  tags: [writing/structure]
  type: judgment
  question: "Does the first paragraph state the main point — not backstory, pleasantries, or setup?"
  applies_when: "User is optimizing posts, emails, memos, or anything read in a scanner's first 3 seconds."
  example_pass: "We're killing the v2 API on March 1. Here's what changes."
  example_fail: "I wanted to reach out because lately I've been thinking about..."
```

### `writing/length`

```yaml
- id: writing-length-range
  tags: [writing/length]
  type: command
  command_template: "wc -w < {target}"
  extract_template: "raw"
  check_template: ">= {min} && <= {max}"
  question: "Is the word count within the target range?"
  applies_when: "User specified a word-count range (always ask explicitly during setup)."
  example_pass: "150 words when range is 100–200"
  example_fail: "350 words when range is 100–200"
```

### `writing/tone`

```yaml
- id: writing-banned-phrases
  tags: [writing/tone]
  type: judgment
  question: "Is the output free of every phrase in the banned list: {banned_list}?"
  applies_when: "User provided bad examples the agent can scan for repeated weak phrasing."
  example_pass: "(no banned phrases appear)"
  example_fail: "This is a game-changer. Here's the kicker: the best part is how we level up."
```

```yaml
- id: writing-no-filler-openers
  tags: [writing/tone]
  type: judgment
  question: "Does the output avoid generic filler openers like 'I hope this email finds you well', 'Just wanted to touch base', 'I was browsing'?"
  applies_when: "User is optimizing any outbound message."
  example_pass: "(no filler openers)"
  example_fail: "I hope this email finds you well. I was browsing LinkedIn and..."
```

## Code patterns

### `code/performance`

```yaml
- id: code-perf-lighthouse-score
  tags: [code/performance]
  type: command
  command_template: "npx lighthouse {url} --output=json --only-categories={category}"
  extract_template: ".categories.{category}.score"
  check_template: ">= {threshold}"
  question: "Does Lighthouse score meet the threshold?"
  applies_when: "User is optimizing a web page they can serve locally."
  example_pass: "0.95"
  example_fail: "0.72"
```

```yaml
- id: code-perf-api-latency
  tags: [code/performance]
  type: command
  command_template: "hey -n 200 -c 10 {url} | awk '/Average:/{print $2*1000}'"
  extract_template: "raw"
  check_template: "< {threshold_ms}"
  question: "Is average API latency under the threshold?"
  applies_when: "User is optimizing a running HTTP endpoint."
  example_pass: "74ms when threshold is 100ms"
  example_fail: "180ms when threshold is 100ms"
```

```yaml
- id: code-perf-cli-runtime
  tags: [code/performance]
  type: command
  command_template: "hyperfine --warmup 3 --export-json /tmp/hf.json '{command}' && jq -r '.results[0].mean' /tmp/hf.json"
  extract_template: "raw"
  check_template: "< {threshold_seconds}"
  question: "Is CLI runtime under the threshold?"
  applies_when: "User is optimizing a CLI tool's execution speed."
  example_pass: "0.42s when threshold is 0.5s"
  example_fail: "1.8s when threshold is 0.5s"
```

### `code/correctness`

```yaml
- id: code-correctness-output-unchanged
  tags: [code/correctness]
  type: judgment
  question: "Does the output match the baseline behavior for a set of reference inputs? Compare against baselines saved before the run."
  applies_when: "Always paired with any code/performance optimization to catch feature removal."
  example_pass: "Output bytewise-identical to baseline."
  example_fail: "Missing fields or different values compared to baseline."
```

### `code/size`

```yaml
- id: code-size-bundle-bytes
  tags: [code/size]
  type: command
  command_template: "du -sk {build_dir} | cut -f1"
  extract_template: "raw"
  check_template: "< {threshold_kilobytes}"
  question: "Is the built bundle under the size threshold?"
  applies_when: "User is reducing bundle size of a built frontend."
  example_pass: "4.2MB when threshold is 5MB"
  example_fail: "7.1MB when threshold is 5MB"
```

## Forecast patterns

### `forecast/consistency`

```yaml
- id: forecast-reconciles
  tags: [forecast/consistency]
  type: judgment
  question: "Do all totals reconcile — revenue sums to ARR, costs sum to total costs, ending cash = starting cash + net cash flow, no orphaned line items?"
  applies_when: "Any financial forecast or model. Always include this eval."
  example_pass: "Every total matches its components; spot-check 3 rows at random."
  example_fail: "Q3 revenue total is 97% of the sum of its monthly values."
```

```yaml
- id: forecast-no-div-zero
  tags: [forecast/consistency]
  type: judgment
  question: "Is the forecast free of #DIV/0, #REF, #NAME, or other formula errors?"
  applies_when: "Spreadsheet-based forecasts."
  example_pass: "No error cells anywhere."
  example_fail: "Cell C47 shows #DIV/0."
```

### `forecast/defensibility`

```yaml
- id: forecast-assumption-in-range
  tags: [forecast/defensibility]
  type: judgment
  question: "Are all key assumptions within defensible market ranges (e.g., monthly churn 1–8%, CAC payback 3–24 months, monthly growth 0–30%)? Flag any assumption outside these ranges with a citation or justification."
  applies_when: "User is optimizing for defensibility, not historical fit."
  example_pass: "All assumptions either within range or explicitly cited."
  example_fail: "Monthly growth assumed at 45% with no justification."
```

```yaml
- id: forecast-assumption-documented
  tags: [forecast/defensibility]
  type: judgment
  question: "Is every key assumption (growth, churn, CAC, pricing, costs) paired with a source, rationale, or 'same as last year' note?"
  applies_when: "Any forecast intended to be reviewed by a stakeholder."
  example_pass: "Each assumption cell has a comment or a named source row."
  example_fail: "Growth rate is 12% with no explanation."
```

### `forecast/fit-to-actuals`

```yaml
- id: forecast-backtest-error
  tags: [forecast/fit-to-actuals]
  type: judgment
  question: "Does the forecast, when run against the last {N} months of actual data, produce predictions within {tolerance}% of actuals on each key line item?"
  applies_when: "User has historical actuals and is optimizing for predictive accuracy."
  example_pass: "Every predicted line within 8% of actuals over last 6 months when tolerance is 10%."
  example_fail: "Revenue prediction 23% off actuals in month 3."
```

## Document patterns

### `docs/completeness`

```yaml
- id: docs-required-sections
  tags: [docs/completeness]
  type: judgment
  question: "Does the document contain every required section: {section_list}?"
  applies_when: "User specifies required sections during setup."
  example_pass: "All of Summary, Problem, Approach, Risks, Timeline are present."
  example_fail: "Missing Risks section."
```

### `docs/citation`

```yaml
- id: docs-claims-cited
  tags: [docs/citation]
  type: judgment
  question: "Is every quantitative or factual claim paired with a specific source (link, document name, or person)?"
  applies_when: "Research docs, memos, strategy papers."
  example_pass: "Every number has a source."
  example_fail: "Three numbers appear without sources."
```

## Visual patterns

### `visual/legibility`

```yaml
- id: visual-text-legible
  tags: [visual/legibility]
  type: judgment
  question: "Is all text in the output fully legible with no truncated, overlapping, or cut-off words?"
  applies_when: "Generated diagrams, slides, images with text."
  example_pass: "Every word complete and readable."
  example_fail: "One label is cut off at the edge."
```

### `visual/palette`

```yaml
- id: visual-palette-constrained
  tags: [visual/palette]
  type: judgment
  question: "Does the output use only colors from the allowed palette: {palette}, with no neon or high-saturation colors outside it?"
  applies_when: "Brand-constrained visual output."
  example_pass: "All colors are from the pastel palette."
  example_fail: "Bright red used in one element."
```

## Prompt patterns

### `prompt/output-shape`

```yaml
- id: prompt-output-structure
  tags: [prompt/output-shape]
  type: judgment
  question: "Does the skill's output match the required structural contract: {contract}?"
  applies_when: "User is optimizing a skill or prompt with a specific output contract."
  example_pass: "Output follows the exact structure defined."
  example_fail: "Output is missing a required section."
```

### `prompt/anti-pattern`

```yaml
- id: prompt-no-numbered-steps
  tags: [prompt/anti-pattern]
  type: judgment
  question: "Is the output free of numbered steps, ordinals, or sequential numbering when the skill explicitly forbids them?"
  applies_when: "User is optimizing a skill with a no-numbering rule."
  example_pass: "No numbers or ordinals appear."
  example_fail: "Output includes '1.', '2.', '3.' sequence."
```
