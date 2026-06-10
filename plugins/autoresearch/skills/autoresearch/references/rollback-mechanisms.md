# Rollback Mechanisms

The autoresearch loop requires the ability to undo a mutation when it makes the target worse. This file documents the four supported mechanisms, how the front-door picks one, and the per-mechanism operations the experiment loop dispatches to.

---

## Derivation tree

Given the target classification from Pass 1 (Classify) of the front-door, pick the mechanism:

```
Text-serializable files, ALL inside one git repo? → git
Text file(s) outside any git repo?                → offer git-init on their common dir, else snapshot-dir
Binary artifact (image, audio, compiled)?         → snapshot-dir
Live system with export/restore API?              → API snapshot
Live system without API?                          → manual-confirm + hard warning
Cannot be reverted at all (sent email)?           → refuse the run
Multiple targets with mixed types?                → most conservative (snapshot-dir)
Targets spread across MULTIPLE git repos?         → snapshot-dir, or ask the user to split the run
```

User can override the proposed mechanism at the confirmation step. Common override: an engineer on a code target who wants snapshot-dir to avoid touching git history.

**One mechanism per run, covering every target.** A rollback that reverts only some target files leaves "discarded" mutations silently alive in the rest — that is worse than no rollback. If the chosen mechanism cannot cover all targets, pick a different mechanism or split the run.

---

## Loop operations (what the experiment loop dispatches to)

The loop in SKILL.md is written mechanism-generically. Each step maps to:

| Loop operation | git | snapshot-dir | API | manual-confirm |
|---|---|---|---|---|
| Verify anchor (step 3) | tag `autoresearch/[name]/good` == HEAD | latest `-good` dir matches targets | last `keep` export matches live state | last confirmed log entry |
| Record attempt (step 4) | commit target files (`--no-verify`) | copy targets to `iterations/NNNN-attempt/` | apply change via API | user applies change, agent logs it |
| Roll back (steps 5/7) | `git reset --hard autoresearch/[name]/good` | copy latest `-good` dir over targets | restore endpoint with kept `export_id` | tell user what to revert, wait for ack |
| Advance anchor (step 7 KEEP) | `git tag -f autoresearch/[name]/good` | copy targets to `iterations/NNNN-good/` | append `keep` entry with new `export_id` | log `keep` entry, user ack |

**The anchor lives forward.** After every KEEP, advance the anchor immediately — the tag, the latest `-good` dir, the most recent kept `export_id`, the last confirmed log entry. Never depend on `HEAD~1` or similar relative references, and never leave the anchor pointing at a pre-keep state: if the run stops right after a keep, a stale anchor makes recovery silently destroy the best result.

---

## Mechanism 1: git (default for code and text-serializable targets)

**Storage:** the run branch `autoresearch/[name]` plus the tag `autoresearch/[name]/good`, advanced to HEAD after each kept experiment. Both are namespaced per run so repeated or concurrent runs cannot clobber each other.

**Rollback operation:** `git -C <target repo> reset --hard autoresearch/[name]/good`.

**Resume after crash:** run `git -C <target repo> status`; if target files are dirty, reset to the tag. The tag advances on every keep, so this never loses kept work.

**Pre-flight checks:**

1. **Resolve the target repo.** For each target file, `git -C <dir-of-target> rev-parse --show-toplevel`. ALL target files must resolve to the SAME repository — call it the target repo, and run every git command as `git -C <target repo>`. If targets span repos or only some are inside one, fall through to snapshot-dir (or ask the user to split the run). Never run git commands against whatever repo happens to enclose the cwd.
2. `git -C <target repo> status --porcelain` returns empty (no uncommitted changes). If not → abort with:
   > "Uncommitted changes detected. Please commit or stash before running autoresearch. I will NOT auto-commit your work."
3. Not on a detached HEAD — the run needs a branch to return to.
4. No collision: the tag `autoresearch/[name]/good`, the branch `autoresearch/[name]`, or the directory `autoresearch-[name]/` already existing means a previous run by that name — offer resume or a new name.

**Setup actions (after pre-flight passes):** create and check out the run branch `autoresearch/[name]`; add `autoresearch-*/` to `.gitignore` and commit that change if it changed; tag HEAD as `autoresearch/[name]/good`. Mutation commits stage ONLY the declared target files and use `--no-verify`.

**User-facing copy (first-mention glossed):**

> *"Rollback: git. I'll work on a separate branch called `autoresearch/[name]` — your branch is never touched. If a change makes your target worse, I'll run `git reset --hard` to throw it away. Your latest good state is saved as a git tag so we can always return to it, and your original files are backed up before anything starts."*

---

## Mechanism 2: snapshot-dir (default for binary artifacts or text files outside a repo)

**Storage:**

```
autoresearch-[name]/
├── baselines/                          # original target files, copied once at setup
│   └── [mirrored relative paths]
└── iterations/
    ├── 0000-baseline/
    ├── 0001-attempt/                   # every attempt, even discarded
    ├── 0001-good/                      # copy of the latest kept state
    ├── 0002-attempt/
    └── 0002-good/                      # moves forward as new kept versions arrive
```

**Setup actions:** copy target files to `iterations/0000-baseline/` and `0000-good/`.

**Record attempt (loop step 4):** after mutating, copy the target files to `iterations/NNNN-attempt/` before running guards.

**Rollback operation:** copy files from the highest-numbered `-good` dir back to the target paths.

**Advance anchor (KEEP):** copy the target files to `iterations/NNNN-good/`.

**Resume after crash:** scan `iterations/`, find the highest-numbered `-good` dir, compare against current target files. If they differ, restore from the `-good` dir.

**Pre-flight checks:**

1. `autoresearch-[name]/` directory is writable.
2. If `autoresearch-[name]/` already exists from a prior run → prompt user: "Resume prior run or start fresh under a new name?" Never overwrite a previous run's `baselines/`.
3. Disk-space check: estimate `size_of_target_files × max_iterations × 2`. If > 25% of available disk, warn the user.

**User-facing copy:**

> *"Rollback: snapshot folder. Before each change I'll copy your current files into `autoresearch-[name]/iterations/NNNN/`. If the change makes things worse, I restore from the last good snapshot. Your original files are saved in `autoresearch-[name]/baselines/` and never touched."*

---

## Mechanism 3: API snapshot (for live systems with export/restore APIs)

**Storage:** `autoresearch-[name]/api-state.json` with one entry per iteration:

```json
{
  "iterations": [
    {
      "id": 0,
      "status": "baseline",
      "export_id": "export_abc123",
      "captured_at": "2026-04-17T14:00:00Z"
    },
    {
      "id": 1,
      "status": "keep",
      "export_id": "export_def456",
      "captured_at": "2026-04-17T14:05:00Z"
    }
  ]
}
```

**Rollback operation:** call the system's restore endpoint with the `export_id` of the most recent kept iteration.

**Advance anchor (KEEP):** export the new state and append it as a `keep` entry.

**Resume after crash:** read `api-state.json`, find the last `keep` status, verify the live system matches (via API read). If mismatched, call restore with that `export_id`.

**Pre-flight checks:**

1. API credentials are present and authenticated (one test call).
2. **Round-trip test:** export the current state → restore it → verify bytewise equality (or API-defined equivalence). If the round-trip fails, **downgrade to manual-confirm** with a warning. This check is non-optional.

**User-facing copy:**

> *"Rollback: API snapshot. Before each change I'll save a snapshot of your {system} through its API. If a change makes things worse, I restore that snapshot. If the snapshot/restore roundtrip doesn't work reliably, I'll stop and tell you — this mode won't proceed silently if it can't undo."*

---

## Mechanism 4: manual-confirm (last resort for live systems without API or irreversible side effects)

**Storage:** `autoresearch-[name]/manual-snapshots.md` — a log the user maintains alongside the agent:

```markdown
## Iteration 0 (baseline)

User snapshot taken: [description of what the user did — screenshot, export, etc.]
User ack: YES at 2026-04-17T14:00:00Z

## Iteration 1 (keep)

Agent proposed change: [description]
User applied change: YES at 2026-04-17T14:05:00Z
User ack rollback-available: YES
```

**Rollback operation:** agent pauses, tells the user what to revert, waits for the user to confirm.

**Pre-flight checks:**

1. Explicit acknowledgment from the user:
   > "I understand the agent cannot automatically revert changes. I will confirm each iteration and manually snapshot before each change."

   No silent manual mode. If the user doesn't ack, abort.

**Note:** manual-confirm is the one sanctioned exception to the loop's "never stop between experiments" rule — the per-iteration pause for the user's snapshot and ack IS the mechanism.

**User-facing copy:**

> *"Rollback: manual. Your {system} doesn't have an API I can use to undo changes automatically. Before each experiment I'll stop and ask you to snapshot the current state yourself (export, screenshot, whatever works). If a change makes things worse, I'll tell you what to revert, and you'll confirm you've done it before we continue. This is slower — expect to spend 30–60 seconds per iteration just on snapshots."*

---

## Refused targets

Some targets can't be undone at all. The agent must refuse to run on these:

- **Sent messages** — emails that have been delivered, SMS, posts to public social media.
- **Irreversible state mutations** — payment processors, physical device actuators, database DROP operations.
- **Systems where a partial revert would be worse than no revert** — distributed systems mid-operation, eventually-consistent stores without point-in-time recovery.

When the agent detects one of these, it must output:

> *"This target can't be safely rolled back — {reason}. Autoresearch requires the ability to undo mutations. Either move to a reversible target (e.g., optimize the template instead of sending the actual messages), or use a different tool."*

---

## Interaction with output modes

- **single-winner, top-N:** one rollback anchor per run. All mechanisms work unmodified. (Top-N finalists are materialized from the run's kept history at delivery — see output-modes.md.)
- **exploration:** every candidate mutates FROM the base anchor, and each kept variant gets its own anchor alongside it:
  - git → tags: `autoresearch/[name]/variant-1`, `autoresearch/[name]/variant-2`, … On KEEP, tag the variant, then `reset --hard` back to the base anchor (`autoresearch/[name]/good`, which stays on the baseline) before the next candidate.
  - snapshot-dir → parallel `iterations/variant-N-good/` directories; targets are restored from `0000-good/` before the next candidate.
  - API → per-variant entries under `variants` in `api-state.json`; restore the baseline export before the next candidate.
  - manual-confirm → per-variant sections in `manual-snapshots.md`.
