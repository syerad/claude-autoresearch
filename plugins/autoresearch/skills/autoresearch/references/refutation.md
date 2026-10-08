# Refutation

Judges answer the questions the run was set up with. A refuter asks the question nobody wrote down: *how is this wrong?* The loop uses one in three places — on the evals before the run, on every would-be keep, and on the final result.

**The rule that makes it safe: a counterexample only counts when it reproduces.** Refuters are tuned to find faults, and near the top of a run they will always find *something* — an objection the judges can't settle measures the refuter's creativity, not the target. So refuter output never changes a score. A finding acts only through a reproducer the loop runs itself, repeatedly, against both versions, and a regression only counts if the earlier version passes every time where the candidate fails every time.

---

## dispatching the refuter

Dispatch the plugin's `autoresearch:refuter` agent: read-only tools, no shell, a fixed output contract. If it isn't available (the skill was copied without its plugin), dispatch the most restricted read-only subagent type available with the contents of the plugin's `agents/refuter.md` as its instructions, and tell the user it isn't tool-restricted.

The brief, per mode:

| Brief contains | `attack-evals` | `gate` | `final` |
|---|---|---|---|
| Mode name | ✓ | ✓ | ✓ |
| The goal — the `goal` paragraph in `config.yaml` | ✓ | ✓ | ✓ |
| Objective, constraints, and guards, verbatim from `config.yaml` | ✓ | ✓ | ✓ |
| Absolute path of the worktree root, and of the target files in it | ✓ | ✓ | ✓ |
| Absolute path of `tasks/` (dev inputs), if any | ✓ | ✓ | ✓ |
| Diversity dimensions, marked as intended differences (exploration) | — | ✓ | ✓ |
| Diff — written to a file under `diffs/` in the artifacts directory; the brief gives its path (the refuter has no shell) | — | anchor → candidate: `git -C <worktree> diff autoresearch/[name]/good autoresearch/[name]` | run base → final: `git -C <worktree> diff <base commit> autoresearch/[name]` |
| Reproducer timeout (the experiment timeout) | — | ✓ | ✓ |

Never put in a brief: the hypothesis, the changelog, `results.tsv`, scores, judge verdicts or reasons, or anything from `holdout/`. A refuter that knows why a change was made argues with the reasoning instead of attacking the result; one that sees held-out data leaks it into promoted constraints.

---

## 1. attack the evals (setup step 12)

At the end of setup, before the baseline, while the user is still present:

1. Dispatch in `attack-evals` mode.
2. `STATUS: HELD` → say so and continue.
3. `STATUS: GAMEABLE` → for each exploit, decide whether a mutation search could plausibly stumble into it. If so, draft a constraint that blocks it — a golden-output `diff`, a required-content check, a judgment — and calibrate a judgment on the labelled examples as in Pass 5. Present each as exploit → proposed constraint, and let the user accept, edit, or reject it. Accepted constraints go into `config.yaml` (still revision 1) and `commands/`, with checksums.

This is the cheapest refutation in the run: one dispatch, before any experiment has been spent climbing a number that can be gamed.

---

## 2. the gate on a would-be keep (loop step 7)

Runs after re-measurement and the held-out check, before KEEP — so only on candidates that are genuinely better by the run's own measure.

1. Dispatch in `gate` mode.
2. `STATUS: HELD` → KEEP.
3. `STATUS: COUNTEREXAMPLES` → **safety-review every reproducer before running it.** It runs unattended in a checkout that shares refs, tags, and stashes with the user's repo. Reject it as not reproduced, without running it, if it invokes `git`; writes anywhere but `/tmp` or `/dev/null`; references `holdout/` or `autoresearch-*/holdout` in any form, or globs over `autoresearch-*/` or its parent; uses `rm`, `mv`, `eval`, or `bash -c`; runs `kill`, `pkill`, or `docker stop|rm`; makes `curl -X POST|PUT|DELETE` requests or uses the network beyond what the run's own evaluators use; uses `sudo`; or references any path outside the worktree and `/tmp` except the run's own `commands/` and `refutations/` directories — or if setup step 2 would have refused it. This is a screen, not a sandbox: it catches the careless case, and the checksums on `commands/` and the tracked-files check after each run catch what it misses.
4. **Reproduce each survivor** 3 times against the candidate and 3 times against the anchor, each run with a fresh experiment-sized timeout, switching state as for re-measurement. After every switch confirm it worked — `git -C <worktree> rev-parse HEAD` equals the intended commit — before running anything, then re-run the guards, so the binary or build being tested is the version you think it is ([rollback-mechanisms.md](rollback-mechanisms.md)):
   - **Command reproducer:** write it to `commands/` and run it as `timeout <remaining>s bash <script>` from the worktree root, with `<remaining>` the fresh experiment-sized timeout. Before each run, note `git -C <worktree> status --porcelain`; after it, restore tracked files (`git -C <worktree> reset --hard <commit under test>`) and delete only the files that weren't there before — build outputs the guards made stay.
   - **Input + question:** each run, a fresh subagent runs the target on the input in the current state, then 3 blind judges get only the output and the question. An output fails when at least 2 of the 3 answer "no".
5. **Classify each:**

   | Candidate (3 runs) | Anchor (3 runs) | Verdict | Effect |
   |---|---|---|---|
   | fails all 3 | passes all 3 | **reproduced** — a regression this change caused | blocks the keep |
   | fails all 3 | fails all 3 | pre-existing | doesn't block; note it in the changelog and in `refutations/pre-existing.md` for the final report |
   | anything else | — | not reproduced (or too flaky to tell) | doesn't block; note it in the changelog |
   | rejected by the safety review, errors, or times out | — | not reproduced | doesn't block; note it in the changelog |

6. **End back on the candidate, on the run branch** (`git -C <worktree> checkout autoresearch/[name]`), whatever the outcome — a keep or a rollback done from the detached anchor state corrupts the run branch.
7. Any reproduced counterexample → status **`refuted`**. Roll back to the anchor, and promote every reproduced counterexample to a constraint:
   - save it under `autoresearch-[name]/refutations/[experiment]-[k].*`, where `k` is the counterexample's number in the refuter's list; a command reproducer is also its script in `commands/`;
   - append it to `constraints` in `config.yaml` with `origin: "refuted experiment N"` — a command reproducer as `command: "bash <absolute path of its script in the artifacts directory>"` with `check: exit 0` (constraints run from the worktree root, so a relative `commands/…` path would not resolve), an input-and-question as a judgment checked the way it was reproduced: 3 judges per run, majority deciding;
   - add its files to `eval_inputs_sha256`.

   It needs no re-baseline: it passed all 3 runs on the anchor, which is the stability check every constraint needs, and constraints never add to the score. Promote it only if a run fits within the experiment timeout; otherwise leave it as a changelog note, since a constraint that times out would discard every later candidate. Bump `revision` and continue.
8. Log the `results.tsv` row with status `refuted` and the candidate's re-measured score; the description names the claim ("… — refuted: drops the Pricing section on vendor comparisons").

Cost: one dispatch plus at most 3 reproducers × 2 states × 3 runs, per would-be keep only.

**Exploration mode:** the same gate, with the base anchor as the earlier version and the diversity dimensions in the brief. A variant that differs along a chosen dimension is doing its job; only a counterexample about something every variant must keep counts. **Manual-confirm runs:** the user makes each state switch, as for a rollback.

---

## 3. the final review (deliver results)

After the loop stops, and after any finalist or variant is materialized:

1. Dispatch in `final` mode with the diff from the run's base (`base.commit` in `config.yaml`) to the final state.
2. Safety-review and reproduce every counterexample as in the gate — 3 runs against the final state and 3 against the base, confirming each switch (`git -C <worktree> rev-parse HEAD` equals the intended commit) and re-running the guards after it (git: `git -C <worktree> checkout --detach <base commit>`, then back with `git -C <worktree> checkout autoresearch/[name]`).
3. Report reproduced regressions prominently in the delivery, before the integration offer: what breaks, the reproducer, and a recommendation not to merge until it is fixed. Don't revert on your own — the user decides. List the pre-existing findings from `refutations/pre-existing.md` separately.
4. If the user has a review agent of their own (a spec or code reviewer), offer to run it on the run branch as well before any merge or PR.

Individual keeps were each gated; the final review exists because changes that each hold on their own can still combine into a regression.
