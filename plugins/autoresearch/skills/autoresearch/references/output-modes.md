# Output Modes

The autoresearch loop supports three output modes. The front-door proposes one during setup based on the user's target type and intent verb; the user can override.

---

## Mode A: single-winner (today's default for optimization)

**When to propose:** User wants "the best" version. Phrasing: "make X faster", "improve Y", "optimize Z", "find/pick the best X". Target types: code performance, skill prompts, bundle size — anything with a clear "hit the threshold" goal.

**Loop behavior:** unchanged. One lineage of kept commits; final state = current state after the loop stops.

**Output:**
- Target files in their final optimized state (on the `autoresearch/[name]` branch for git runs).
- Score summary: baseline % → final %.
- Top 3 most impactful changes from the changelog.
- Remaining failure patterns (what still fails).

**Rollback:** single anchor — the `autoresearch/[name]/good` tag (git) or latest `-good` dir (snapshot).

---

## Mode B: top-N (deliver finalists for the user to choose from)

**When to propose:** User asked for "options", "a few variants", "A/B candidates", "some choices", or wants to "compare" directions. Target types: marketing copy, UI microcopy, email templates, pitch lines.

**Loop behavior:** unchanged during iteration — same mutate/guard/evaluate/keep-or-discard loop. Difference is only at the end.

**Selection needs per-eval data.** The total score in `results.tsv` is not enough to compute domination. In top-N mode, every evaluated experiment also appends its per-eval pass counts to `autoresearch-[name]/scores.json`:

```json
{
  "experiments": [
    {"id": 1, "status": "keep", "commit": "abc1234",
     "per_eval": {"Opening specificity": 5, "Single concrete ask": 4, "Length 40-80w": 5, "No banned phrases": 5}}
  ]
}
```

**Selection at end of loop:**

1. Take all kept experiments from `scores.json`.
2. Filter to the **non-dominated** set: variant A dominates B iff for all evals E, per_eval_A(E) >= per_eval_B(E) and for some eval E*, per_eval_A(E*) > per_eval_B(E*). Keep every variant nothing dominates.
3. If > N non-dominated variants, rank by total pass-rate and take the top N.
4. If < N, return what was found.

**Output:**
- Side-by-side diff of the N finalists (rendered in terminal and in dashboard).
- Per-finalist eval score vector (one row per eval, one column per finalist), straight from `scores.json`.
- Prompt: "Which one do you want as the working copy?"
- **Materializing the pick (git):** restore the finalist's target files onto the run branch — `git -C <target repo> checkout <finalist commit> -- <target files>`, commit as `autoresearch: select finalist [K]`, and advance the anchor tag. No history rewriting; later kept commits remain in the branch history. (Snapshot-dir: copy the finalist's `-good` dir over the targets and record a new `-good`.)

**Rollback:** same as single-winner during the loop. Selection is post-hoc.

---

## Mode C: exploration (portfolio of distinct valid variants)

**When to propose:** User wants to "explore scenarios", "see the possibilities", or is working on a target where "one winner" is the wrong shape. Target types: financial forecasts, pricing strategies, product positioning, research directions, strategy options. Forecast/pricing/strategy target types default to exploration even if the user's verb is ambiguous.

**Loop behavior:** star-shaped instead of linear. The baseline is the hub:

- **Every candidate mutates FROM the base anchor** (the baseline state), not from the last kept variant. This keeps variants comparable on the diversity dimensions and prevents drift compounding across variants.
- A new candidate is kept only if it passes three tests:
  1. All guards pass.
  2. All evals pass (in exploration mode, evals are hard constraints — not a score).
  3. **Differs from every existing kept variant on at least one diversity dimension.**
- **On KEEP (git):** tag the commit `autoresearch/[name]/variant-K`, then `reset --hard` back to the base anchor before the next candidate. (Other mechanisms: see rollback-mechanisms.md "Interaction with output modes".)
- **On DISCARD:** roll back to the base anchor as usual.

**The baseline counts.** The baseline is variant `base` — it must pass all evals at setup (abort the run if it doesn't; exploration requires a valid starting point), and it counts toward N. "N=3" means the base plus two new kept variants, unless the user explicitly asks for N new variants.

**Diversity dimension (required for exploration mode):**

During setup, the agent asks:

> "What makes two variants meaningfully different? Pick 1–3 dimensions."

Examples by target type:

- Financial forecast: growth rate, churn, CAC, pricing model, cost structure
- Pricing: price tier count, free-trial length, discount cadence
- Product positioning: target persona, core promise, competitive frame
- Copy A/B at scale: opening hook shape, CTA shape, length tier

For each chosen dimension the agent also asks for a "meaningful difference" threshold:

- Categorical (e.g., pricing model ∈ {flat, tiered, usage-based}) → any different value counts
- Numeric (e.g., growth rate) → default 10% relative delta, editable

A new variant is considered "different" from an existing one if any one chosen dimension shows a difference meeting the threshold.

**No diversity dimension → run top-N instead.** Distinctness cannot be measured against the eval set: kept variants all pass every eval by rule 2, so their pass/fail patterns are identical and no second variant could ever qualify — the loop would burn its whole budget returning one variant. If the user can't name a dimension, that's the signal their goal is "give me good options", which is exactly top-N. Degrade to top-N and tell them why.

**Stop condition:**

- N distinct valid variants reached (counting `base`), OR
- Iteration budget exhausted.

If the budget is exhausted with fewer than N variants, return what was found with a note: "Found {k} of {N} requested variants within the budget."

**Logging:** every experiment still goes to `results.tsv` (kept variants use status `keep` with a `variant K — …` description) and the changelog; the dashboard's `variants` array carries the gallery data and `variants_target` carries N.

**Output:**

- Gallery of the kept variants (dashboard's variant-gallery view).
- Per-variant "what's different" summary:
  > "Variant 2 (bear case): churn 4.5% vs. base 2.8%; CAC payback 22mo vs. 14mo. Everything reconciles."
- No single "winner" — user picks which variant(s) to keep or export. To materialize one as the working copy, restore its tagged state the same way top-N materializes a finalist.

**Rollback:** per-variant anchors around a fixed base anchor. See `rollback-mechanisms.md` "Interaction with output modes".

---

## Mode selection decision table

| User phrasing or context | Proposed mode |
|---|---|
| "make X faster / better / smaller" | single-winner |
| "optimize X for Y" | single-winner |
| "find the best X" / "pick the best X" | single-winner |
| "give me some options for X" | top-N |
| "a few variants of X" | top-N |
| "A/B candidates for X" | top-N |
| "compare approaches / directions for X" | top-N |
| "explore scenarios" | exploration |
| "bull / base / bear" | exploration |
| "what are the possibilities" | exploration |
| Financial forecast, pricing, or strategy target type | exploration (overrides ambiguous verb) |
| Exploration requested but no diversity dimension nameable | top-N (see Mode C) |

Always present the proposed mode to the user with a one-sentence rationale and an explicit override path:

> "Based on your goal I'd run in **exploration mode** — 3 distinct variants instead of one winner. Say 'single-winner' or 'top-3' to change."

---

## Per-mode dashboard behavior

See `dashboard-template.html` for implementation. Data model is backwards-compatible: `mode: "single"` (or absent) produces the score chart; `mode: "top-n"` adds a finalist row at the end; `mode: "exploration"` adds the variant gallery (with `variants_target` as the progress denominator) while keeping the experiments table — every mode logs experiments.
