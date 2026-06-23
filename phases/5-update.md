# Phase 5 — Update existing docs

Bring an existing documentation set back in sync with code that has changed since
the docs were last touched. Use this phase when the docs already exist and the
user wants them refreshed, not when creating docs from scratch (that is Phase 1).

## Resuming from cold context

To start this phase, read:

- `.scaffold-docs.yml` — the metadata file: audience, last-synced SHA, flags.
  If it does not exist, see "Missing metadata" below.
- `overall-structure.md` — the doc set's intended structure and topic inventory.
- The existing docs: `getting-started.md`, `diving-deeper/`, `reference/`.
- The codebase being documented.

If no docs exist at all, this is the wrong phase. Direct the user to Phase 1
(`phases/1-discovery.md`).

## 5a. Recover the audience

Read `audience` from `.scaffold-docs.yml`.

- **Found** → use it as the governing lens for every change in this phase, exactly
  as the creation flow does. Read `references/audience-check.md` now and apply its
  three questions throughout this phase.
- **Not found, `ask_audience: true`** (or no metadata file) → offer the user two
  paths, and act on their choice:
  1. *Define an audience now.* Run a trimmed Phase 1b: propose 3 candidate primary
     audiences from the codebase, the user picks one, then propose up to 3
     secondaries. Write the result to `.scaffold-docs.yml` with `audience.inferred: false`.
     Then apply `references/audience-check.md` as above.
  2. *Skip it.* Set `ask_audience: false` in `.scaffold-docs.yml` so this phase
     never prompts about audience again. Then proceed with the style anchor below.
- **Skipped, or `ask_audience: false`** → do not prompt. Derive a **style anchor**
  from the existing docs: their voice, depth, and the knowledge they already
  assume of the reader. Edit new and changed content to match that style so the
  docs stay internally consistent. The style anchor replaces the audience lens for
  this run; do not apply `references/audience-check.md`'s benefit-framing question,
  since there is no audience to frame for.

### Missing metadata

If `.scaffold-docs.yml` does not exist (docs were written before this feature, or
by hand), treat the audience as "not found" above. For the sync baseline, use the
fallback in 5b. At the end of the phase (5e), write a fresh `.scaffold-docs.yml`
so the next run has a clean baseline.

## 5b. Establish the change baseline

Find the commit the docs were last reconciled to, then diff code against it.

1. **Baseline = `last_synced_sha`** from `.scaffold-docs.yml` when present. This is
   authoritative.
2. **Fallback** when there is no metadata or no SHA: the doc files' last git
   commit. Find it with, e.g.,
   `git log -1 --format=%H -- getting-started.md diving-deeper reference`.
3. Diff the documented repo from the baseline to `HEAD`
   (`git diff <baseline>..HEAD --stat` and full diff as needed).

The two signals cross-check each other: if the doc files' last commit is newer
than `last_synced_sha` but touched only docs (a typo fix, say), still diff code
from `last_synced_sha`. A doc-only commit must not reset the baseline.

## 5c. Classify changes by significance

Sort the diff into three groups and present them to the user:

- **Significant** — warrants doc updates: new, removed, or renamed public exports;
  changed signatures; new modules; behavior changes evidenced in comments or
  tests.
- **Not significant** — skip: internal refactors, formatting, private helpers,
  doc-only commits.
- **Borderline** — do not silently decide. List these for the user and let them
  choose whether each warrants a doc change.

If nothing is significant (and the user dismisses any borderline items), say so
and stop. Do not invent busywork. If you stop here, offer to advance the baseline
anyway: ask the user whether to update `last_synced_sha` in 5e to the current
`HEAD`, so the next run starts clean.

## 5d. Map changes to buckets, then walk each bucket

First, an internal mapping step (not a checkpoint): for each significant change,
note which buckets and files it touches.

- New public API → Reference (plus a Diving Deeper topic if the *intent* needs
  explaining).
- Changed core workflow → Getting Started.
- New design decision → Diving Deeper.
- Removed or renamed symbol → all three, for accuracy.

Then walk the buckets **in this order, one at a time**: Getting Started →
Diving Deeper → Reference. Getting Started goes first: it is the highest-value
bucket and best handled while attention is fresh.

For each bucket:

1. Propose the specific edits for that bucket — quote the existing text and show
   the proposed replacement, governed by the audience lens (or style anchor).
2. **CHECKPOINT.** The user reviews, edits, and approves.
3. Apply the approved edits to the files. Show file paths.
4. Move to the next bucket.

Do not batch all buckets into one upfront report. Each bucket is its own
checkpoint.

## 5e. Record the new sync point

After the last bucket is approved and applied, update `.scaffold-docs.yml`:

- `last_synced_sha` → the current `HEAD` SHA of the documented repo.
- `last_synced_date` → today's date.
- If you created the file this run (missing metadata), write the full schema,
  including the recovered or defined `audience` and the current `ask_audience`
  value.

Show the file path. The docs are now in sync, and the next update run has a clean
baseline.
