---
name: spec-prepare
description: |
  Use this skill when the user invokes /spec-prepare with a feature slug whose plan.md is
  already plan-approved. It is step 3 of the spec-driven chain (spec-design → spec-plan
  → spec-prepare → spec-implement) that replaces /implement-task. The skill reads plan.md
  and requirements.md, resolves the ADO project and area path from CLAUDE.md (or asks
  if missing), asks the user to create or provide an existing ADO Feature to group the
  PBIs under, pulls that Feature's own acceptance criteria from ADO and checks the
  drafted PBI set as a whole against them before approval, detects existing PBIs for the
  slug before creating, creates one PBI per plan item (stories, enablers, spikes) with
  Title/Description/Acceptance Criteria, sets predecessor links to model the dependency
  graph, and writes pbis.md as a local mirror — preserving any already-resolved ADO ID
  across resumed or reopened runs rather than re-blanking it. Re-invoking after approval
  is treated as an explicit, confirmed reopen that warns about downstream ADO items and
  implementations before proceeding. Ends only on explicit user approval.
  Trigger phrase: "/spec-prepare <feature-slug>"
version: 2.0.0
argument-hint: <feature-slug>
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Edit
  - Agent
  - AskUserQuestion
  - TodoWrite
  - mcp__azure-devops__wit_create_work_item
  - mcp__azure-devops__wit_get_work_item
  - mcp__azure-devops__wit_get_work_item_type
  - mcp__azure-devops__wit_get_work_items_batch_by_ids
  - mcp__azure-devops__wit_update_work_item
  - mcp__azure-devops__wit_work_items_link
  - mcp__azure-devops__wit_add_child_work_items
  - mcp__azure-devops__wit_query_by_wiql
  - mcp__azure-devops__core_list_projects
  - mcp__azure-devops__search_workitem
---

# spec-prepare

Step 3 of the spec-driven chain. Translate an approved `plan.md` into real Azure DevOps
PBIs, grouped under a user-provided Feature, with predecessor links modelling the
dependency graph. Writes `.specs/<slug>/pbis.md` as a local mirror. **No code is
written.** Only `pbis.md` and `_meta.md` may be created or edited.

## Invocation

```
/spec-prepare <feature-slug>
```

## Chain context

1. `/spec-design` → `requirements.md`
2. `/spec-plan` → `plan.md`
3. **`/spec-prepare`** → ADO PBIs + `pbis.md` (this skill)
4. `/spec-implement <pbi-id>` → branch, commits, PR (replaces `/implement-task`)

Each step is invoked explicitly. This skill never auto-runs `/spec-implement`; it ends
by printing the exact next command.

## Cross-cutting rule: self-critique via parallel agents

At every step below where Claude is forming an opinion, drafting content, or about to
present something to the user, Claude **must** spawn **2–3 Agent calls in parallel**
with adversarial prompts attacking its own current thinking. Examples:

- "Find PBI descriptions that are too vague to implement without re-reading plan.md."
- "Find predecessor links that are missing, wrong, or create a cycle."
- "Find AC that duplicates something already in a different PBI — or that was silently
  dropped from the plan."

Critique findings are folded back into the work **before** presenting to the user.

## Steps

### 1. Preconditions

Read `.specs/<feature-slug>/_meta.md`.

- **Missing** → `Run /spec-design <slug> first.` Stop.
- **`status` is `prepare-in-progress`** → an earlier run of this skill (fresh or reopened) was
  interrupted before final approval. **Resume:** load the existing `pbis.md` draft and
  continue from wherever it left off — normally Step 9's approval gate. Do not restart the
  draft from Step 2.
- **`status` is exactly `plan-approved`** → normal fresh run. Continue to Step 2.
- **`status` is earlier than `plan-approved`** (`design-in-progress`, `design-approved`) →
  the plan hasn't been approved yet. Tell the user and print `Run /spec-plan <slug> first.`
  Stop.
- **`status` is later than `plan-approved`** (`prepare-approved`, `implement-in-progress`,
  `implement-done`) → this is a **reopen**, not a fresh run — PBIs, and possibly
  implementations, already exist for the plan this run is about to redraft. Follow the
  Reopen protocol below before doing anything else.

**Reopen protocol.**

1. **Show what's downstream.** Read `pbis.md` (which plan items already carry real ADO
   IDs) and `implementation.md` if it exists (which items are already implemented, on
   which branches). Present both before asking anything.
2. **Confirm explicitly.** `AskUserQuestion`: *"PBIs — and possibly implementations —
   already exist for this plan. Reopening lets you redraft PBI content, but anything
   already created in ADO or already implemented may go stale relative to the new draft.
   Continue?"* — **Reopen** (explicit) or **Cancel** (leave everything as is; stop).
3. **Roll back status.** On confirmed reopen, set `_meta.md status: prepare-in-progress` and
   proceed to Step 2 as normal. **Do not skip Step 5** — it is exactly what re-syncs the
   new draft against what already exists in ADO without destroying it (see Steps 5 and 7).
4. **On approval (Step 13), restore `status: prepare-approved`** and note in `pbis.md`'s header
   that this is a reopened draft with the date, so the history isn't silently overwritten.

Read `.specs/<feature-slug>/plan.md` and `.specs/<feature-slug>/requirements.md`.
If either is absent, treat as a corrupted state and ask the user how to proceed.

**Spec validation preflight.** Verify plan.md has the required sections and structure:
- `## Ordered execution list` — at least one item present.
- `## Dependency graph` — present (may be empty for a single-item plan).
- `## Items` — at least one item section with a valid YAML front-matter block containing
  all required fields: `id`, `type`, `title`, `deps`, `hours`, `refs`, `narrative`,
  `ac_behavioral`, `ac_nonbehavioral`.
If any required section or YAML field is missing, report the specific issue and ask
the user to run `/spec-plan <slug>` to repair it before continuing.

### 2. Parse plan.md

Extract and hold in working memory the following structures. Every downstream step
draws from this parsed representation — never re-reads the file ad-hoc.

**2a. Ordered execution list**
The numbered sequence from the "Ordered execution list" section. This is the creation
order for Step 10 and the basis for the `/spec-implement` command list in Step 13.

**2b. Per-item record** (one per story `S-*`, enabler `E-*`, spike `K-*`)
From the YAML front-matter block of each item:
- `id` — plan item ID (e.g. `S-1`)
- `type` — `story` | `enabler` | `spike`
- `title` — short imperative title (becomes the ADO Title field)
- `deps` — list of plan item IDs this depends on (used in Step 11)
- `hours` — integer estimate (appended to ADO Description)
- `refs` — FR/NFR/AD identifiers (appended as traceability line in ADO Description)
- `narrative` — story sentence / why-enabler exists / spike question (from YAML field)
- `ac_behavioral` — list of GWT scenario strings (from YAML field)
- `ac_nonbehavioral` — list of non-behavioral bullet strings (from YAML field)

The YAML block is always the FIRST thing in each item section. All fields are required;
empty lists are valid. The YAML schema is the canonical contract — do not parse
unstructured prose for AC or narrative content.

**2c. Dependency graph**
Build a lookup map from the YAML `deps` fields (authoritative) and cross-check against
the prose "Dependency graph" section for consistency:
`{ plan-item-id → [list of prerequisite plan-item-ids] }`.

If the YAML `deps` and prose adjacency list disagree, YAML wins. Surface any
discrepancy to the user before continuing.

**2d. Additional Code Surface** (if present)
Record the section for passing to `/spec-implement` later. Written into `pbis.md` as a
reference section.

Each Code Surface entry follows this machine-readable schema:
- `path` — repo-root-relative path (e.g. `src/exports/CsvExporter.cs`)
- `reason` — one-line rationale for why this file is relevant
- `confidence` — `high` | `medium` | `low`
- `greenfield` — `true` | `false` (`true` = projected intent, not existing code)

Entries without a `path` field or with `path = null` are skipped. The Starting Points
sub-list is an ordered subset of these entries.

**2e. Open Questions carried forward** (if present)
If any have status `accepted-open`, note them — they may indicate AC that is
intentionally incomplete. Surface them to the user before PBI creation in a single
message: list each question, its owning item ID, and ask: *resolve now / convert to
K-item / accept as documented risk*. Do not proceed until every accepted-open question
has a disposition.

**Self-critique here.** Spawn **2 parallel Agent calls**:
1. "Are there items in the ordered execution list that don't have a corresponding YAML
   block, or YAML blocks that don't appear in the execution list?"
2. "Are there `deps` entries that reference an ID not present in the item list
   (dangling dependency)?"
Resolve any mismatches by surfacing them to the user before continuing.

### 3. Resolve ADO project and area path

Read `CLAUDE.md` in the project root. Look for keys such as:
- `ado-project`, `azure-devops-project`, `project`
- `ado-area-path`, `area-path`, `team`

Also check `_meta.md` — if a prior skill run wrote `ado-project` or `ado-area-path`
there, use those values directly without re-resolving.

If found in either source, confirm with a single short message to the user.

If **not found**, call `AskUserQuestion`:
- "What is the Azure DevOps project name?"
- "What area path should these PBIs be created under?"

Write the confirmed values into `_meta.md` (`ado-project`, `ado-area-path`) so
downstream skills inherit them without re-resolving.

**ADO field preflight.** Call `mcp__azure-devops__wit_get_work_item_type` for work item
type `Product Backlog Item` in the resolved project. Confirm that the field
`Microsoft.VSTS.Common.AcceptanceCriteria` exists in the process template. If it does
not, warn the user and ask for the correct field reference name before continuing.

Do not proceed until project, area path, and AC field name are all confirmed.

### 4. Resolve the parent Feature

The user creates the ADO Feature manually (or provides an existing one).

Show the user:
- The feature slug and the title from `requirements.md §1 Context`.
- A suggested Feature title: `<Title from Context>`.

Then call `AskUserQuestion`:
> "Please create an ADO Feature in project `<project>` under area `<area>` with the
> title above (or provide an existing Feature ID if one already exists).
> Paste the Feature work item ID when ready."

After receiving the Feature ID, call `mcp__azure-devops__wit_get_work_item` and confirm:
- Item exists
- WorkItemType is `Feature`
- It is in the correct project and area path

While fetching, also capture `Microsoft.VSTS.Common.AcceptanceCriteria` from the response
— the Feature's own acceptance criteria, if any. Hold it for Step 8; do not re-fetch it
there unless the draft is revised in a later run and a fresh read is warranted.

If validation fails, report the specific error and ask the user to provide a corrected ID.

Wait for a validated Feature ID before continuing. All PBIs created in Step 10 will be
linked as children of this Feature.

### 5. Detect existing PBIs

Build a **per-item resolution record** now — `{ plan item ID → disposition, real ADO ID
(if any) }` — and carry it forward unchanged into Step 7 and Step 10. This record is what
makes redrafting `pbis.md` safe on a resumed or reopened run: **once an item's real ADO ID
is known, nothing downstream may blank it back to a placeholder.**

Before creating anything, use `pbis.md` as the **primary duplicate registry**: if
`pbis.md` already exists and contains a non-`—` ADO ID for a plan item, that item is
**already resolved** — disposition `already-created`, with that real ID. Skip ADO
querying for it entirely.

For plan items not already resolved, search ADO by the feature slug tag.
All PBIs created by this skill are tagged with the feature slug (e.g. `csv-export`).
Query by tag for reliable, injection-safe duplicate detection:

```wiql
SELECT [System.Id], [System.Title], [System.WorkItemType]
FROM WorkItems
WHERE [System.TeamProject] = '<project>'
  AND [System.WorkItemType] = 'Product Backlog Item'
  AND [System.Tags] CONTAINS '<feature-slug>'
ORDER BY [System.CreatedDate] DESC
```

If matches are found, present them to the user grouped by plan item. For each match ask:
- **Update** — overwrite Title, Description, and AC with the plan content. Disposition
  `update`, with the matched real ADO ID.
- **Skip** — leave the existing PBI untouched. Disposition `skip`, with the matched real
  ADO ID.
- **Create new** — ignore the match and create a fresh PBI. Disposition `new`, no ID yet.

Plan items with no match anywhere get disposition `new`, no ID yet.

Record the resolution record per item before moving to Step 6.

### 6. Draft PBI content

For each plan item (stories `S-*`, enablers `E-*`, spikes `K-*`) that will be created
or updated, draft the three ADO fields from the parsed YAML (Step 2b):

**Title** — taken directly from the YAML `title` field, unmodified.
- For spikes (`K-*`): prefix with `[Spike]` (e.g. `[Spike] Investigate auth token refresh`).

**Description** — composed from the YAML `narrative` field plus:
- Append a `Traceability` line: `Refs: FR-1, NFR-2, AD-3` (from YAML `refs`).
- Append an `Estimated effort: <hours>h` line (from YAML `hours`).

**Acceptance Criteria** — composed from `ac_behavioral` and `ac_nonbehavioral` YAML
fields, formatted as plain text for the ADO rich-text field:
- Behavioral: each `ac_behavioral` entry as a `Given / When / Then` paragraph.
- Non-behavioral: each `ac_nonbehavioral` entry as a bullet.

**Tags** — every PBI receives the feature slug as a tag (e.g. `csv-export`).
Spikes (`K-*`) also receive the tag `Spike`. Tags are comma-separated in ADO.

**Self-critique here.** Before writing the draft, spawn **2 parallel Agent calls**:
1. "Find any PBI description that is too vague — i.e., an implementer could not start
   without re-reading plan.md."
2. "Find ac_behavioral or ac_nonbehavioral items from plan.md that did not make it into
   any PBI's drafted AC."
Revise before moving to Step 7.

### 7. Write draft `pbis.md` (local only — no ADO upload yet)

Write `.specs/<feature-slug>/pbis.md` as a **local draft**, rendering each item from its
Step 5 resolution record — never from a blanket "every item is new" assumption:

| Disposition (Step 5) | ADO ID column | Status column |
|---|---|---|
| `new` | `—` | `draft` |
| `update` | the matched real ID | `draft (update pending)` |
| `skip` | the matched real ID | `skipped (existing)` |
| `already-created` | the real ID already in `pbis.md` | `created` |

**The ADO ID column always carries the real ID once one is known — never a placeholder
with the ID mentioned only in prose elsewhere.** This is the column Step 10 and
`/spec-implement` both read to resolve an item; burying an ID in the Status column breaks
both. If this is a fresh run, every item is `new`. If this is a resume or a reopen (Step
1), some items already carry real IDs from Step 5 — **preserve them exactly**; redrafting
`pbis.md` must never regress a resolved ID back to `—`.

```markdown
# <Feature Name> — PBIs  [DRAFT — not yet uploaded to ADO]

**ADO Project:** <project>
**Area Path:** <area path>
**Feature ID:** <ADO Feature work item ID provided by user>

## PBI Map

| Plan ID | ADO ID | Type | Title | Status |
|---------|--------|------|-------|--------|
| S-1 | — | PBI | <title> | draft |
| E-1 | — | PBI | <title> | draft |
| K-1 | — | PBI (Spike) | [Spike] <title> | draft |
| S-2 | 12290 | PBI | <title> | skipped (existing) |
…

## Predecessor links to set
- S-2 → depends on → E-1
- E-1 → depends on → S-1
…

## Additional Code Surface
(copied from plan.md if present — for /spec-implement hand-off)

## PBI Details

### S-1 — <title>
**Type:** Product Backlog Item
**Refs:** FR-1, NFR-2
**Estimated effort:** 12h

**Description:**
<drafted description text>

**Acceptance Criteria:**
<drafted AC text>

### K-1 — [Spike] <title>
**Type:** Product Backlog Item
**Tags:** Spike
…
```

The "PBI Details" subsection is the full content that *will* be uploaded — what the user
approves here is exactly what lands in ADO.

### 8. Feature acceptance criteria coverage check

The Feature grouping these PBIs may carry its own acceptance criteria — a coarser
statement of what the whole slice of work must deliver, sitting above what any single
PBI's own AC proves. Before the draft goes to approval, check that the PBI set **as a
whole** covers it. This is not optional and it is not the same thing as Step 6's per-PBI
AC drafting — that step makes sure each PBI's AC traces back to `plan.md`; this step makes
sure the *union* of all PBIs traces up to the Feature.

**Use the Feature AC captured in Step 4.** Do not re-fetch unless this is a later run
against a revised draft and a fresh read is warranted — in that case, re-call
`mcp__azure-devops__wit_get_work_item` for the Feature ID and re-read
`Microsoft.VSTS.Common.AcceptanceCriteria`.

- **Feature AC is empty** → record "Feature carries no acceptance criteria of its own" in
  `pbis.md` and continue. Nothing to check.
- **Feature AC is present** → parse it into discrete criteria (bullets, numbered items, or
  sentences — whatever shape the ADO field returns) and check each one against the union
  of every drafted PBI's title, description, and acceptance criteria (Step 6 output, all
  items together — not one PBI at a time).

Classify each Feature-level criterion:
- **Covered** — at least one PBI's AC or description clearly addresses it.
- **Partially covered** — related work is drafted but the criterion isn't fully satisfied
  by what's there.
- **Not covered** — no drafted PBI touches it at all.

**Self-critique here.** Spawn **2 parallel Agent calls**:
1. "Read the Feature's acceptance criteria and the full drafted PBI set. Find every
   Feature AC item that is not actually covered by any PBI — be skeptical of loose
   paraphrase; a PBI that's merely adjacent in topic doesn't count as coverage."
2. "Argue the opposite: find a PBI the classification credited with covering a Feature AC
   item that it doesn't really address, or only addresses partially."
Reconcile findings into the classification above before presenting anything.

**Any `not covered` or `partially covered` item blocks the approval gate (Step 9).**
Present each gap to the user with `AskUserQuestion` and three dispositions:
1. **Extend an existing PBI's AC** — the work is already drafted, it just wasn't stated
   explicitly; update that PBI's Description or Acceptance Criteria in the draft.
2. **Reopen planning** — the gap is real, uncovered work; stop and print
   `Run /spec-plan <feature-slug>` to add the missing story/enabler, then re-run
   `/spec-prepare <feature-slug>`. Do **not** fabricate a new plan item from inside this
   skill — every PBI must trace back to a `plan.md` item (Step 2b), and creating one here
   would break that invariant.
3. **Accept as out of scope** — explicit user confirmation; recorded as a documented gap,
   never silently dropped.

Do not proceed to Step 9 until every Feature AC item has a disposition.

Write the result into `pbis.md` as a new section, placed after "Additional Code Surface":

```markdown
## Feature acceptance criteria coverage
Feature <ID>: <title>

| Feature AC item | Status | Covered by | Notes |
|---|---|---|---|
| <criterion text> | covered | S-1, E-2 | |
| <criterion text> | accepted-gap | — | out of scope per user, <date> |
```

If the Feature carries no AC of its own, write a one-line note instead of the table:
`Feature <ID> carries no acceptance criteria of its own — nothing to check.`

### 9. Approval gate — review draft before any ADO upload

Present `pbis.md` to the user. This is the **only** point at which content can be
changed before it reaches ADO.

Invite edits:
- Change a title, description, or AC item → edit `pbis.md` and re-present.
- Remove an item → mark it `skipped` in the PBI Map.
- Change the Feature ID → update the header.
- Reorder → update both the PBI Map and the "Predecessor links to set" section;
  re-validate topological order against the `deps` graph after reordering.

**Self-critique before each presentation.** Spawn **2 parallel Agent calls**:
1. "Read this draft pbis.md and find anything that doesn't faithfully represent the
   plan's ordered execution list or dependency graph."
2. "Find plan items missing from the PBI map, or PBI details sections with no
   corresponding map entry, or a Feature AC coverage item left without a disposition."
Revise before showing.

**Exit condition for this gate.** Proceed to Step 10 only when the user explicitly says
"approved", "upload", "go", "looks good", "done", or "ship it".
Implicit silence is not approval.

### 10. Create / update PBIs in ADO

**Re-read `.specs/<slug>/pbis.md` from disk** as the first action. Do not use any
in-memory representation from earlier steps. The user may have edited the file during
the Step 9 approval loop; the disk version is authoritative.

Work through the **approved** `pbis.md` PBI Map top-to-bottom (execution order), acting
according to each item's Status column:
- `draft` → create.
- `draft (update pending)` → update, using the real ID already in the ADO ID column.
- `skipped (existing)` or `created` → already settled in ADO; make no call for this item.

**Create** (status `draft`):
```
mcp__azure-devops__wit_create_work_item
  project: <project>
  type: "Product Backlog Item"        // stories, enablers, AND spikes
  title: <drafted title>              // spikes: "[Spike] <title>"
  description: <drafted description>
  Microsoft.VSTS.Common.AcceptanceCriteria: <drafted AC>
  Microsoft.VSTS.Scheduling.OriginalEstimate: <hours>   // from YAML hours field
  areaPath: <area path>
  tags: "<feature-slug>"              // ALL items — enables duplicate detection on re-run
        "<feature-slug>; Spike"       // K-items — both slug tag AND Spike tag
```

**Update** (status `draft (update pending)`):
```
mcp__azure-devops__wit_update_work_item
  id: <existing PBI id>
  title: <drafted title>
  description: <drafted description>
  Microsoft.VSTS.Common.AcceptanceCriteria: <drafted AC>
```

**Link to Feature** (for every created or updated PBI):
```
mcp__azure-devops__wit_work_items_link
  sourceId: <Feature ID>
  targetId: <PBI ID>
  linkType: "System.LinkTypes.Hierarchy-Forward"
```

**Write the real ADO ID to `pbis.md` immediately** after each successful create or
update call, replacing the `—` placeholder for that specific item. Do not batch — write
after each individual call. This makes `pbis.md` an accurate recovery point if the
process is interrupted.

If any ADO call fails: stop immediately, report the error and the last successfully
created ADO ID, and ask the user to resolve the issue before re-running. On re-run,
Step 5's pbis.md check will skip already-created items automatically.

### 11. Set predecessor links in ADO

Using the dependency graph from the **approved** `pbis.md` "Predecessor links to set"
section, set one link per `deps` entry using the real ADO IDs from Step 10.

For "A depends on B" (B must complete before A), set:
- `sourceId` = B (the prerequisite — must finish first)
- `targetId` = A (the dependent — waits for B)
- `linkType` = `"System.LinkTypes.Dependency-Forward"`

This encodes: B → A in ADO's board view (B is the predecessor of A).

> **Note:** Two things need confirming against the `mcp__azure-devops__wit_work_items_link`
> tool schema before the first call, not just assumed from this doc — getting either wrong
> produces no error, just a silently inverted or mislabeled graph in ADO:
> 1. The exact parameter name (`relationType` vs `linkType`).
> 2. Which of `Dependency-Forward` / `Dependency-Reverse` actually means "predecessor" from
>    `sourceId`'s side, per the tool's own documentation or a one-item test call. Do not
>    trust the mapping asserted above without checking it.

**Self-critique here.** Spawn **2 parallel Agent calls**:
1. "Check these predecessor links against the approved pbis.md dependency section.
   List any missing links or links that create a cycle."
2. "Are there implicit dependencies between PBIs that the plan didn't name explicitly
   but that obviously exist (e.g., a story that writes to a schema an enabler creates)?"
Resolve discrepancies before continuing.

### 12. Finalise `pbis.md`

By this point every ADO ID should already be written into `pbis.md` from Step 10.
Final clean-up:
- Update any remaining `draft` → `created`, `draft (update pending)` → `updated`.
  `skipped (existing)` and `created` (from `already-created`) needed no ADO call and stay
  as they were.
- Append actual ADO URLs next to each ADO ID.
- Remove the `[DRAFT]` header marker.
- Update the "Predecessor links to set" section heading to "Predecessor links set".

```markdown
# <Feature Name> — PBIs

**ADO Project:** <project>
**Area Path:** <area path>
**Feature ID:** 99001 — [<Feature title>](<ADO Feature URL>)

## PBI Map

| Plan ID | ADO ID | Type | Title | Status |
|---------|--------|------|-------|--------|
| S-1 | 12345 | PBI | <title> | created |
| E-1 | 12346 | PBI | <title> | created |
| K-1 | 12347 | PBI (Spike) | [Spike] <title> | created |
| S-2 | 12348 | PBI | <title> | skipped (existing) |
…

## Predecessor links set
- S-2 (12348) → depends on → E-1 (12346)
- E-1 (12346) → depends on → S-1 (12345)
…
```

### 13. Final confirmation

Present the finalised `pbis.md` as a done summary. No further approval needed — the
user already approved the content in Step 9 before upload.

1. Update `_meta.md`: `status: prepare-approved`, add `prepare_approved: <today's date>`.

   Full allowed `status` values for the chain (for reference):
   - `design-in-progress` — /spec-design running
   - `design-approved` — requirements.md approved
   - `plan-in-progress` — /spec-plan running
   - `plan-approved` — plan.md approved
   - `prepare-in-progress` — /spec-prepare running
   - `prepare-approved` — PBIs uploaded and confirmed
   - `implement-in-progress` — /spec-implement running on at least one PBI
   - `implement-done` — all PBIs implemented, PR(s) created
2. Print the exact next commands (one per PBI, in execution order):

   ```
   Done. To implement, run (one PBI per invocation, in this order):
     /spec-implement <ADO-ID-of-S-1>
     /spec-implement <ADO-ID-of-E-1>
     …
   ```

## Behaviour rules

- **Write only `.specs/<slug>/pbis.md` and `.specs/<slug>/_meta.md`.** Never edit
  `requirements.md` or `plan.md`.
- **Never create a PBI without checking for duplicates first** (Step 5).
- **Never upload to ADO before the draft `pbis.md` is explicitly approved** (Step 9).
- **Every Feature AC item must have a disposition before approval** (Step 8) — covered,
  extended into an existing PBI, sent back to `/spec-plan`, or explicitly accepted as a
  gap. Never silently drop a Feature-level criterion.
- **Never blank a resolved ADO ID back to a placeholder.** Once Step 5 records a real ID
  for an item, every later draft of `pbis.md` (including a resumed or reopened run) must
  render that ID, in the ADO ID column, exactly as resolved.
- **Reopening after `prepare-approved` or later requires explicit confirmation** (Step 1) and
  must show what's downstream (existing ADO IDs, existing implementations) before asking.
- **Write each ADO ID to `pbis.md` immediately after each successful create** — never
  batch at the end.
- **Never auto-run `/spec-implement`.** Always wait for the user.
- **Every plan item must appear in `pbis.md`**, even if skipped — gaps are bugs.
- **Predecessor links must faithfully reflect `plan.md`'s dependency graph** — YAML
  `deps` is authoritative, not the prose adjacency list.
- **Every PBI is tagged with the feature slug** — this is what makes duplicate
  detection reliable on re-runs. Never create a PBI without setting the slug tag.
