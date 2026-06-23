# Design: `update-docs` mode for scaffold-docs

**Date:** 2026-06-22
**Status:** Approved (pending implementation plan)

## Problem

`scaffold-docs` creates a three-tier documentation set (Getting Started, Diving
Deeper, Reference) for a code library. But repos keep changing after the docs are
written. There is no supported way to bring stale docs back in sync with the code.
Two gaps make this hard today:

1. **No record of when docs were last synced to code.** Nothing tracks the point
   in history the docs were reconciled to, so there is no reliable baseline for
   "what changed since."
2. **The audience is not durably stored for reuse.** The primary/secondary
   audience lives at the top of `overall-structure.md`, mixed into Phase-1
   planning content. It can be edited or lost, and nothing else reads it.

The whole skill is audience-governed: every pass applies
`references/audience-check.md` against the primary audience. An update flow must
recover that lens or operate sensibly without it.

## Goal

Add an **update mode** to the existing skill (not a separate skill) that:

- Determines what code changed since the docs were last touched.
- Recovers the target audience, or handles its absence gracefully.
- Walks the existing docs bucket by bucket and proposes section-level changes.
- Applies changes with user feedback at each bucket.
- Records the new sync point so the next run has a clean baseline.

## Non-goals

- Re-running the full structure → headers → prose pass sequence for unchanged
  sections. Updates are incremental.
- Producing a single large upfront "change report" before any edits. The flow
  walks one bucket at a time instead.
- Automatic, unattended doc rewriting. The user stays in the loop per bucket.

## Design

### 1. Where it lives

- New phase file: `phases/5-update.md`.
- `SKILL.md` is updated so the skill activates on update-intent
  (e.g. "update the docs," "the docs are stale," "sync the docs with the code"),
  and the workflow overview references the new phase.
- The existing creation flow gains one new responsibility: write the metadata
  file (§2) when it finishes. A repo documented before this feature existed will
  have no metadata file; update mode handles that (§5, §6).

### 2. The metadata file

A dedicated, human-readable file in the docs root: **`.scaffold-docs.yml`**,
sitting next to `overall-structure.md`. YAML so a user can hand-edit the audience
or flip a flag.

```yaml
last_synced_sha: <git SHA the docs were last reconciled to>
last_synced_date: 2026-06-22
audience:
  primary: <label + 1–2 sentence description>
  secondary:
    - <label — description>
  inferred: false        # true if derived from docs, not user-chosen
ask_audience: true       # false = user told us to stop asking about audience
```

`overall-structure.md` remains Phase-1 planning content. Operational state moves
here so edits to one file do not clobber the other. This file is the single
source of truth for update mode.

### 3. Detecting what changed (both signals, cross-checked)

1. **Baseline = `last_synced_sha`** from metadata when present. Authoritative.
2. **Fallback** when metadata is missing or has no SHA: the doc files' last git
   commit.
3. **Significance filter:** diff `HEAD` against the baseline, then classify
   changes.
   - *Significant:* new/removed/renamed public exports, changed signatures, new
     modules, and behavior changes evidenced in comments or tests.
   - *Not significant:* internal refactors, formatting, private helpers, and
     doc-only commits. A doc-only commit (e.g. a typo fix) must **not** reset the
     baseline.
   - *Borderline:* surface these to the user and let them decide rather than
     silently judging.

The two signals cross-check each other: if the doc files' last commit is newer
than `last_synced_sha` but touched only docs, update mode still diffs code from
`last_synced_sha`.

If there are no significant code changes, the command says so and stops rather
than inventing busywork.

### 4. The review flow — walk each bucket independently

1. **Internal mapping step (not a checkpoint):** map each significant code change
   to the buckets and files it affects.
   - New public API → Reference (plus Diving Deeper if intent needs explaining).
   - Changed core workflow → Getting Started.
   - New design decision → Diving Deeper.
   - Removals/renames → all three, for accuracy.
2. **Bucket by bucket, in order: Getting Started → Diving Deeper → Reference.**
   For each bucket the agent proposes the specific edits, the user gives feedback,
   the agent applies them, then moves on. Each bucket is its own checkpoint. There
   is no single large upfront report. Getting Started goes first because it is the
   highest-value bucket and best handled while attention is fresh.
3. **On completion:** update `last_synced_sha` and `last_synced_date` in the
   metadata file to the reconciled commit.

### 5. The audience branch

On entry, recover the audience from metadata.

- **Found** → use it as the governing lens, exactly as the creation flow does
  (`references/audience-check.md` applies throughout).
- **Not found, `ask_audience: true`** → offer two paths:
  1. *Define one now* — a trimmed Phase 1b: propose 3 candidate primary
     audiences, the user picks one (and optional secondaries), written to
     metadata.
  2. *Skip it* — set `ask_audience: false` so update mode never prompts about
     audience again.
- **Skipped, or `ask_audience: false`** → no formal audience. The agent derives a
  **style anchor** from the existing docs (voice, depth, assumptions already on
  the page) and edits to match it, keeping new content consistent with old.

### 6. Edge cases

- **Missing metadata file:** fall back to doc files' last commit for the baseline
  (§3.2) and run the audience branch as "not found" (§5).
- **No docs present at all:** update mode is the wrong entry point; direct the
  user to the creation flow.
- **No significant changes:** report and stop (§3).
- **Borderline-only changes:** present them and let the user choose what, if
  anything, to update (§3.3).

## Alternatives considered

- **Single upfront triage report** mapping all changes to all buckets before any
  edits. Rejected in favor of walking buckets independently, for a more granular
  flow that keeps each bucket's review focused.
- **Re-run the full pass structure** (structure → headers → prose) scoped to
  changed sections. Rejected as too heavyweight for incremental updates.
- **Store state in `overall-structure.md` or per-doc frontmatter.** Rejected for a
  dedicated metadata file: a single source of truth that survives edits to the
  planning or content files.

## Open implementation questions

- Exact trigger phrasing to add to `SKILL.md`'s `description` so update-intent
  activates the skill without over-triggering on creation-intent.
- Whether the creation flow should backfill `.scaffold-docs.yml` for repos
  documented before this feature, on first update run.
