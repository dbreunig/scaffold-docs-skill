# update-docs Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an update mode to the scaffold-docs skill that brings stale docs back in sync with code, governed by a stored audience and a recorded sync point.

**Architecture:** A new phase file `phases/5-update.md` defines the update workflow. A dedicated `.scaffold-docs.yml` metadata file (written by the creation flow, read by update mode) holds the audience, the last-synced git SHA, and an `ask_audience` flag. `SKILL.md` and `README.md` are updated so update-intent activates the skill and the new file/flow are discoverable.

**Tech Stack:** Markdown skill files. No code, no test harness. Verification is done by re-reading content and grepping for cross-reference and naming consistency.

---

## File Structure

- **Create:** `phases/5-update.md` — the update workflow (change detection, audience recovery, bucket-by-bucket review).
- **Modify:** `phases/1-discovery.md` — step 1e also writes `.scaffold-docs.yml`.
- **Modify:** `SKILL.md` — description triggers on update-intent; workflow overview, output structure, and how-to-use mention the new phase and metadata file.
- **Modify:** `README.md` — document the update command and metadata file.

These four files are the entire surface. Each task below produces one self-contained, committable change.

**Conventions to follow (verify before editing):**
- Phase files open with `# Phase N — Title`, then a `## Resuming from cold context` section listing files to read, then content sections, with `**CHECKPOINT.**`/`## CHECKPOINT` markers at user-review points. (See `phases/1-discovery.md` and `phases/4-reference.md`.)
- The metadata filename is **`.scaffold-docs.yml`** — spelled exactly this way everywhere. The repo is the source of truth; grep for it after each edit to catch drift.

---

### Task 1: Define the metadata file in the creation flow

Make the creation flow write `.scaffold-docs.yml` so update mode has something to read. This is the schema's single point of definition.

**Files:**
- Modify: `phases/1-discovery.md` (step 1e and the CHECKPOINT)

- [ ] **Step 1: Read the current file**

Read `phases/1-discovery.md` in full so the edit lands in the right place (step 1e begins at "## 1e. Produce overall-structure.md").

- [ ] **Step 2: Add a metadata-writing subsection after step 1e's file list**

In `phases/1-discovery.md`, immediately after the `## 1e. Produce overall-structure.md` section's bullet list (the list ending with "Reference items: list of modules...") and before the headline-style instruction paragraph, insert:

```markdown
### Also write the metadata file

Alongside `overall-structure.md`, write `.scaffold-docs.yml` in the same
directory. This is the durable, machine-readable record the update flow
(`phases/5-update.md`) reads later. It is the single source of truth for the
audience and the sync point; `overall-structure.md` stays human planning content.

```yaml
last_synced_sha: <current git HEAD SHA of the documented repo>
last_synced_date: <today's date, YYYY-MM-DD>
audience:
  primary: <primary label — 1–2 sentence description>
  secondary:
    - <label — description>
  inferred: false        # false here: the user chose this audience in 1b
ask_audience: true
```

If the repo is not a git repository, omit `last_synced_sha` and note it; the
update flow falls back to the doc files' last commit.
```

- [ ] **Step 3: Mention the metadata file in the CHECKPOINT**

In the `## CHECKPOINT` section of `phases/1-discovery.md`, change the first line from:

```markdown
Save the file, show its path, and ask the user to review and edit. They should actively decide:
```

to:

```markdown
Save both files (`overall-structure.md` and `.scaffold-docs.yml`), show their paths, and ask the user to review and edit. They should actively decide:
```

- [ ] **Step 4: Verify consistency**

Run: `grep -rn "scaffold-docs.yml" phases/1-discovery.md`
Expected: at least two hits (the new subsection and the CHECKPOINT line), filename spelled `.scaffold-docs.yml` exactly.

Re-read the edited 1e section and confirm the YAML block is well-formed and `inferred: false` / `ask_audience: true` are present.

- [ ] **Step 5: Commit**

```bash
git add phases/1-discovery.md
git commit -m "Write .scaffold-docs.yml metadata in creation flow"
```

---

### Task 2: Create the update phase file

The core deliverable: the full update workflow.

**Files:**
- Create: `phases/5-update.md`

- [ ] **Step 1: Create the file with the full content below**

Create `phases/5-update.md` containing exactly:

````markdown
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
  as the creation flow does. Read `references/audience-check.md` now and apply it
  throughout, as in Phase 2 and Phase 3.
- **Not found, `ask_audience: true`** (or no metadata file) → offer the user two
  paths, and act on their choice:
  1. *Define an audience now.* Run a trimmed Phase 1b: propose 3 candidate primary
     audiences from the codebase, the user picks one, then propose up to 3
     secondaries. Write the result to `.scaffold-docs.yml` with `inferred: false`.
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
and stop. Do not invent busywork. Still update the sync point in 5e if the user
wants the baseline advanced.

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
````

- [ ] **Step 2: Verify structure and naming**

Run: `grep -n "CHECKPOINT" phases/5-update.md`
Expected: at least one CHECKPOINT marker (in 5d).

Run: `grep -c "scaffold-docs.yml" phases/5-update.md`
Expected: several hits, filename spelled `.scaffold-docs.yml` exactly.

Re-read the file and confirm: it opens with `# Phase 5 — Update existing docs`, has a `## Resuming from cold context` section, and references `references/audience-check.md` for the found-audience path.

- [ ] **Step 3: Commit**

```bash
git add phases/5-update.md
git commit -m "Add Phase 5 update-docs workflow"
```

---

### Task 3: Wire the update mode into SKILL.md

Make the skill activate on update-intent and make the new phase and metadata file discoverable in the skill's own navigation.

**Files:**
- Modify: `SKILL.md` (frontmatter description, Output structure, Workflow overview, How to use this skill)

- [ ] **Step 1: Read SKILL.md**

Read `SKILL.md` in full to locate the exact strings below before editing.

- [ ] **Step 2: Extend the frontmatter description to cover update-intent**

In `SKILL.md`, append this sentence to the end of the `description:` field (after "applied during every prose pass."), keeping it on the same YAML line:

```
 Also use to update or refresh existing docs after code changes — phrases like "update the docs", "the docs are stale", or "sync the docs with the code" — which runs the update workflow in Phase 5.
```

- [ ] **Step 3: Note the metadata file in Output structure**

In the `## Output structure` section, after the `- **Reference**: ...` bullet, add:

```markdown
- **Metadata**: `.scaffold-docs.yml`. Machine-readable record of the audience, the last-synced git SHA, and the `ask_audience` flag. Written by Phase 1, read by Phase 5 to refresh docs after code changes.
```

- [ ] **Step 4: Add Phase 5 to the Workflow overview**

In the `## Workflow overview` fenced block, add a line after the Phase 4 line:

```
Phase 5: Update existing docs after code changes    → phases/5-update.md          [USER REVIEW per bucket]
  (read .scaffold-docs.yml → baseline diff → classify changes → walk buckets)
```

- [ ] **Step 5: Note Phase 5 entry conditions in How to use this skill**

In the `## How to use this skill` section, after the paragraph that begins "Each phase has its own file under `phases/`. **Load only the phase you're currently working on.**", add:

```markdown
**Updating existing docs?** When the docs already exist and the user wants them refreshed after code changes, start at Phase 5 (`phases/5-update.md`) instead of Phase 1. Phase 5 reads `.scaffold-docs.yml` to recover the audience and the last-synced commit.
```

- [ ] **Step 6: Verify consistency**

Run: `grep -n "phases/5-update.md\|scaffold-docs.yml\|Phase 5" SKILL.md`
Expected: the description, Output structure, Workflow overview, and How-to-use edits all present; `phases/5-update.md` and `.scaffold-docs.yml` spelled exactly.

Confirm the frontmatter still parses: the `description:` value remains a single unbroken line.

- [ ] **Step 7: Commit**

```bash
git add SKILL.md
git commit -m "Wire Phase 5 update mode into SKILL.md"
```

---

### Task 4: Document the update command in README.md

**Files:**
- Modify: `README.md` (What it does, Usage)

- [ ] **Step 1: Read README.md**

Read `README.md` in full to locate the insertion points.

- [ ] **Step 2: Describe updating in "What it does"**

In `README.md`, after the paragraph ending "You will spend most of *your* time working on **Getting Started**. That's where agents need the most help.", add:

```markdown
### Keeping docs in sync

Code keeps changing after the docs are written. The skill records the audience and
the commit the docs were last synced to in a `.scaffold-docs.yml` file. When you
ask it to update or refresh the docs, it diffs the code since that commit, sorts
the changes by whether they warrant doc edits, and walks the Getting Started,
Diving Deeper, and Reference sections one at a time, proposing changes for your
review in each.
```

- [ ] **Step 3: Show the update usage**

In the `## Usage` section, after the existing fenced `/scaffold-docs` block, add:

```markdown
To refresh existing docs after the code has changed, ask the skill to update them:

\```
/scaffold-docs update the docs
\```

If no audience was recorded (for docs written before this feature, or by hand),
the skill offers to define one with you, or to stop asking and match the existing
docs' style instead.
```

(Write the fenced block with real triple backticks — the `\`` escaping above is only so this plan renders.)

- [ ] **Step 4: Verify**

Run: `grep -n "scaffold-docs.yml\|update the docs\|Keeping docs in sync" README.md`
Expected: the "What it does" subsection and the Usage block both present.

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "Document update-docs mode in README"
```

---

### Task 5: Whole-skill consistency self-check

No new files. Verify the four edited/created files agree with each other and with the spec.

**Files:**
- Review only: `SKILL.md`, `README.md`, `phases/1-discovery.md`, `phases/5-update.md`

- [ ] **Step 1: Filename consistency across the skill**

Run: `grep -rn "scaffold-docs.yml" SKILL.md README.md phases/`
Expected: every hit spells the file `.scaffold-docs.yml` (leading dot, `.yml` not `.yaml`). Fix any drift.

- [ ] **Step 2: Phase 5 is reachable from navigation**

Run: `grep -rn "phases/5-update.md" SKILL.md`
Expected: referenced in both the Workflow overview and the How-to-use section.

- [ ] **Step 3: Audience handling matches the spec**

Re-read `phases/5-update.md` section 5a against the spec
(`docs/superpowers/specs/2026-06-22-update-docs-mode-design.md`, §5). Confirm all
three branches exist: found → audience lens; not found + `ask_audience: true` →
offer define-or-skip; skipped/`ask_audience: false` → style anchor. Confirm the
`ask_audience` flag name matches the schema in `phases/1-discovery.md` exactly.

- [ ] **Step 4: Change-detection matches the spec**

Re-read `phases/5-update.md` sections 5b–5c against spec §3. Confirm: stored SHA is
authoritative, doc-files'-last-commit is the fallback, the significance filter has
significant/not-significant/borderline groups, and borderline items are surfaced
to the user rather than silently judged.

- [ ] **Step 5: Commit any fixes**

```bash
git add -A
git commit -m "Consistency fixes across update-docs mode files"
```

If Step 1–4 found nothing to fix, skip the commit and note the plan is complete.

---

## Self-Review (performed during planning)

**Spec coverage:**
- Spec §1 (where it lives) → Task 2 (phase file), Task 3 (SKILL.md wiring).
- Spec §2 (metadata file) → Task 1 (schema + creation-flow write), referenced in Tasks 2–4.
- Spec §3 (change detection, both signals, significance/borderline) → Task 2 sections 5b–5c; verified in Task 5 Step 4.
- Spec §4 (walk buckets, GS first, record sync point) → Task 2 sections 5d–5e.
- Spec §5 (audience branch, three cases) → Task 2 section 5a; verified in Task 5 Step 3.
- Spec §6 (edge cases: missing metadata, no docs, no significant changes, borderline-only) → Task 2 "Missing metadata", 5a, 5b fallback, 5c stop condition.
- Spec "open implementation questions" (trigger phrasing; backfill metadata) → Task 3 Step 2 sets the trigger phrasing; Task 2 section 5e backfills `.scaffold-docs.yml` on missing metadata.

**Placeholder scan:** No "TBD"/"handle edge cases"/"similar to Task N". Full file content is inline for the created file and every edit shows the exact string.

**Naming consistency:** `.scaffold-docs.yml` and `ask_audience` are used identically in Tasks 1, 2, 3, 4 and checked in Task 5. `phases/5-update.md` named identically throughout.
