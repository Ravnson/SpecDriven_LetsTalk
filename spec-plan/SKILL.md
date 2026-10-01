---
name: spec-plan
description: |
  Use this skill when the user invokes /spec-plan with a feature slug whose
  requirements.md is already design-approved. It is step 2 of the spec-driven chain
  (spec-design → spec-plan → spec-prepare → spec-implement) that replaces /implement-task.
  The skill reads requirements.md, blocks on any unresolved Open Questions until the
  user explicitly approves carry-over, re-challenges the requirements via parallel
  adversarial agents (and may send the user back to /spec-design on a real spec gap),
  produces user stories as vertical slices of user-visible value plus first-class
  technical enablers/spikes, writes acceptance criteria as Given/When/Then with
  bulleted criteria for non-behavioral items, ensures every NFR lands as AC on a
  relevant story, orders work by dependencies → risk → value, estimates in hours, and
  writes plan.md with YAML front-matter per item plus a dependency graph. Re-invoking after
  approval is treated as an explicit, confirmed reopen that warns about downstream PBIs and
  implementations and never renumbers an item ID already referenced by them. Ends only on
  explicit user approval.
  Trigger phrase: "/spec-plan <feature-slug>"
version: 1.2.0
argument-hint: <feature-slug>
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Edit
  - Agent
  - WebSearch
  - WebFetch
  - AskUserQuestion
  - TodoWrite
  - mcp__azure-devops__search_wiki
  - mcp__azure-devops__search_workitem
---

# spec-plan

Step 2 of the spec-driven chain. Transform an approved `requirements.md` into a rigorous
`plan.md` of user stories + technical enablers + spikes, with acceptance criteria,
dependencies, and hour estimates. **No code is written.** Only `.specs/<slug>/plan.md`
and `.specs/<slug>/_meta.md` may be created or edited.

## Invocation

```
/spec-plan <feature-slug>
```

## Chain context

1. `/spec-design` → `requirements.md`
2. **`/spec-plan`** → `plan.md` (this skill)
3. `/spec-prepare <slug>` → ADO PBIs + `pbis.md`
4. `/spec-implement <pbi-id>` → branch, commits, PR (replaces `/implement-task`)

Each step is invoked explicitly. This skill never auto-runs `/spec-prepare`; it ends by
printing the exact next command.

## Cross-cutting rule: self-critique via parallel agents

At every step below where Claude is forming an opinion, drafting content, or about to
present something to the user, Claude **must** spawn **2–3 Agent calls in parallel** with
adversarial prompts that attack its own current thinking. Examples:

- "Read this draft plan.md and list every missed dependency, hidden coupling, and
  optimistic estimate."
- "Argue that story X is actually two stories. Or that two stories should merge."
- "Find NFRs from requirements.md that did not land as AC on any story."
- "Challenge the ordering: is there a cheaper path that reduces risk faster?"

Critique findings are folded back into the work **before** presenting anything to the
user. Each step below names where to invoke it.

## Steps

### 1. Preconditions

Read `.specs/<feature-slug>/_meta.md`.

- **Missing** → tell the user the design step was never run. Print:
  `Run /spec-design <feature-slug> first.` Stop.
- **`status` is `plan-in-progress`** → an earlier run of this skill (fresh or reopened)
  was interrupted before final approval. **Resume:** load the existing `plan.md` draft and
  continue from wherever it left off — normally Step 9's review loop. Do not restart from
  Step 2.
- **`status` is exactly `design-approved`** → normal fresh run. Continue to Step 2.
- **`status` is earlier than `design-approved`** (`design-in-progress`) → the design isn't
  approved yet. Tell the user and print `Run /spec-design <feature-slug> first.` Stop.
- **`status` is later than `design-approved`** (`plan-approved`, `prepare-in-progress`,
  `prepare-approved`, `implement-in-progress`, `implement-done`) → this is a **reopen**, not a
  fresh run — a plan, and possibly PBIs and implementations, already exist. Follow the
  Reopen protocol below before doing anything else.

**Reopen protocol.**

1. **Show what's downstream.** Read `pbis.md` (which plan items already have real ADO
   IDs) and `implementation.md` if it exists (which items are already implemented, on
   which branches). Present both before asking anything.
2. **Confirm explicitly.** `AskUserQuestion`: *"PBIs — and possibly implementations —
   already exist for this plan. Reopening lets you amend it, but item IDs already
   referenced downstream cannot be renumbered or removed (see the ID-stability rule
   below), and anything already created or implemented may go stale relative to the new
   content. Continue?"* — **Reopen** (explicit) or **Cancel** (leave everything as is;
   stop).
3. **ID stability.** Load the existing `plan.md` as the starting point, not a blank slate.
   Steps 3–8 may add new items (continuing the `S-`/`E-`/`K-` numbering) or amend the
   content of existing items **in place**, but must never renumber, reuse, or remove an
   item ID that already appears in `pbis.md` or `implementation.md`. If a change is big
   enough that an existing item's ID no longer fits what it represents, treat the old item
   as superseded (note it as such in the item's body) and add a new item — never repoint
   an ID that downstream files already reference.
4. **Roll back status.** On confirmed reopen, set `_meta.md status: plan-in-progress` and
   proceed to Step 2 as normal.
5. **On approval (Step 9), restore `status: plan-approved`** and note in `plan.md`'s
   Overview that this is a reopened revision, with the date.

Read `.specs/<feature-slug>/requirements.md`. If it does not exist alongside an
approved `_meta.md`, treat as a corrupted state and ask the user how to proceed.

**Spec validation preflight.** Before proceeding, verify requirements.md has all
required sections. Check for the presence of headings: `## 0. Document History`,
`## 1. Context`, `## 2. Goals`, `## 3. Non-Goals`, `## 4. Functional Requirements`,
`## 5. Non-Functional Requirements`, `## 6. Architectural Decisions`, `## 7. Risks`,
`## 8. Open Questions`, `## 9. Glossary`, `## 10. References`, `## 11. Code Surface`.
Also verify: at least one FR exists with a `[SHALL]`/`[SHOULD]`/`[MAY]` qualifier.
If any required section is missing or malformed, report the specific issue and ask
the user to run `/spec-design <slug>` to repair it before continuing.

### 2. Handle Open Questions from requirements.md

Locate the "Open Questions" section of requirements.md.

- If **empty** → continue to Step 3.
- If **non-empty** → **block by default.** List the open questions back to the user with
  three options for each:
  1. **Resolve now** — answer in chat; the answer is folded into the plan and noted
     under "Open Questions carried forward" with status `resolved-in-plan`.
  2. **Carry forward (explicit approval)** — user explicitly approves carrying this
     question into plan.md as a flagged risk. Recorded with status `accepted-open`.
  3. **Reopen design** — user wants to fix this in requirements.md first. Skill stops
     and prints `Run /spec-design <feature-slug>` to revise.

Use `AskUserQuestion`. Skill only proceeds when **every** open question has one of
those three dispositions. Implicit silence is not approval.

**Termination rules:**
- If a question receives 'I don't know' twice in a row, convert it automatically to a K-item (spike) without further prompting. The spike question is the original open question.
- Total spikes created from open questions is capped at 3. If the cap is reached, any remaining unresolved questions are recorded as documented open risks in plan.md and the skill proceeds.
- If the user wants more than 3 spikes, they must explicitly type 'allow more spikes' to raise the cap.

### 3. Re-ingest context — Code Surface first

**Read `requirements.md` section 11 (Code Surface) as ground truth.** That section
already records touched files, reusable code, existing tests, conventions, and
starting points. Do **not** re-discover from zero.

Then in parallel, *extend* (do not replace) that map:

1. **Explore agent — codebase, planning lens.** Starting from the Code Surface
   "starting points", look for things relevant to *implementation cost and coupling*
   that the design phase may have missed: hidden dependencies, downstream callers of
   changed code, shared utilities used across affected files, integration points.
2. **Explore agent — local docs.** `CLAUDE.md`, build/test instructions, deployment
   notes. Constraints that affect ordering and estimates.
3. **ADO search.** Related work items and wiki pages that may indicate prior plans,
   abandoned approaches, or in-flight work to coordinate with.

**Self-critique here.** Spawn 1 parallel Agent call: "Is the existing Code Surface
stale, incomplete, or wrong for the *planning* lens? List anything missing." Fold its
findings into the extension set.

Record any **new code surface** discovered (files, reusable code, tests, conventions
not already in requirements.md section 11) in a working list. This list will become
the "Additional Code Surface" section of plan.md in Step 8. **Never write to
requirements.md.** If the additions are large enough to suggest the spec is wrong,
treat as a spec gap in Step 4.

**Concurrent change isolation check.** Scan all other `.specs/*/` folders for
`_meta.md` files with `status` not equal to `implement-done`. For each active spec,
compare its `claimed_paths` list against the Code Surface paths for this feature
(§11 + any new surface found above). If overlap is found, report it:
- Which other slug claims the same path(s).
- Whether the overlap is in a touched file (high risk) or a read-only dependency (lower risk).
Surface to the user as a warning before Step 4. Do not block — the user decides
whether to coordinate, defer, or accept the risk.

Summarize findings briefly before Step 4, including the Code Surface delta and any
concurrent ownership warnings.

### 4. Re-challenge requirements.md

**Self-critique here is mandatory.** Spawn **3 parallel Agent calls**:

1. "Read requirements.md and find FRs/NFRs that are *unplannable* as written — too
   vague to estimate, missing acceptance shape, or contradicting another requirement."
2. "Argue that requirements.md is missing at least one story that the FRs imply but
   never state explicitly."
3. "Find NFRs that will be silently dropped if not converted into concrete AC."

Merge findings.

If a **true spec gap** is found (not just rephrasable in plan.md), surface it to the
user with three options:
- Fix it inline in plan.md as an assumption + risk.
- Reopen design: stop and print `Run /spec-design <feature-slug>` to revise
  requirements.md.
- Accept the gap as an Open Question carried forward (explicit user approval).

Skill proceeds only after the user picks.

### 5. Draft the item list

Identify items in three buckets:

- **Stories** — vertical slices of user-visible value. May bundle multiple FRs. Each
  must deliver something a user can perceive end-to-end.
- **Technical enablers** — non-user-facing work that must exist for stories to ship:
  schema migrations, new modules, infrastructure changes, refactors that unblock.
- **Spikes** — time-boxed investigations that resolve an unknown before a dependent
  story can be estimated honestly.

For each item, assign using this locked YAML schema:

```yaml
id: S-1                        # plan item ID
type: story                    # story | enabler | spike
title: <short imperative title>
deps: []                       # list of plan item IDs this depends on
hours: 12                      # integer hour estimate
refs: [FR-1, NFR-2, AD-3]      # requirement IDs from requirements.md
narrative: >                   # story sentence / why-enabler / spike question
  As a <role> I want <capability> so that <outcome>.
ac_behavioral:                 # Given/When/Then scenarios (list of strings)
  - "Given … When … Then …"
  - "Given … When … Then …"
ac_nonbehavioral:              # non-behavioral bullets (list of strings)
  - "p95 response time < 500ms under 1k concurrent users (NFR-1)"
```

The YAML block is always the FIRST thing in each item section, before any prose. The YAML schema is the canonical contract consumed by /spec-prepare. All fields are required; empty lists are valid.

**NFR coverage gate (mandatory, blocks plan.md write).** Before writing plan.md:
1. Extract every NFR-N id from requirements.md §5.
2. Collect the union of all refs arrays across every draft item.
3. Diff: any NFR-N not present in the union is an uncovered NFR.
4. For each uncovered NFR, present it to the user and require one of:
   - Assign it to an existing item (update that item's refs and AC).
   - Create a new enabler item specifically for this NFR.
   - Accept it as out-of-scope (explicit user confirmation, recorded as a risk).
5. Do not proceed until every NFR has a disposition.

### 6. Write acceptance criteria

For each story and enabler:

- **Given / When / Then** for every behavioral criterion.
- **Bulleted checklist** for non-behavioral criteria (perf targets, audit/log
  requirements, telemetry, accessibility checks). These often come straight from NFRs.

Spikes get a different shape: a **question to answer** and a **definition of done**:
decision recorded in plan.md as an assumption note appended to the K-item's body, under a ## Outcome heading. If the outcome requires a formal Architectural Decision change, stop and instruct the user to run /spec-design <slug> to revise requirements.md explicitly. The K-item's YAML narrative field is updated to summarise the outcome.

### 7. Determine order

Order: **dependencies first, then risk, then priority, then value.**

1. Topologically sort by `deps` (YAML field is authoritative).
2. Among items with no remaining unmet dependencies, prefer the one whose unknowns are
   highest (spike or risky enabler).
3. Among items of equal risk, prefer the one covering `[SHALL]` FRs over `[SHOULD]`
   over `[MAY]` (read from the `refs` field — look up each FR/NFR's RFC 2119 qualifier
   in requirements.md §4 and §5).
4. Among items of equal risk and priority, prefer highest user-visible value.

Render the result as an explicit ordered list (1, 2, 3, …) and a dependency graph
(adjacency list is fine).

**Self-critique here.** Spawn **2 parallel Agent calls**: one arguing the order is
wrong (cheaper path exists), one arguing a hidden dependency is missing. Revise.

### 8. Draft `plan.md`

Write `.specs/<feature-slug>/plan.md` with this structure:

```markdown
# <Feature Name> — Plan

## Overview
One paragraph: scope, totals (count of stories / enablers / spikes, sum of hours),
links back to requirements.md.

## Ordered execution list
1. S-1 — <title>  (12h)
2. E-1 — <title>  (4h)
3. S-2 — <title>  (8h)
…

## Dependency graph
*Read-only view derived from each item's YAML `deps` field. YAML is authoritative —
if this section and the YAML disagree, YAML wins.*
- S-1: (none)
- E-1: depends on S-1
- S-2: depends on E-1

## Items

### S-1 — <title>
\`\`\`yaml
id: S-1
type: story
title: <short imperative title>
deps: []
hours: 12
refs: [FR-1, FR-2, NFR-1]
narrative: >
  As a <role> I want <capability> so that <outcome>.
ac_behavioral:
  - "Given … When … Then …"
  - "Given … When … Then …"
ac_nonbehavioral:
  - "p95 response time < 500ms under 1k concurrent users (NFR-1)"
  - "Audit log entry written for every export (NFR-3)"
\`\`\`

### E-1 — <title>
\`\`\`yaml
id: E-1
type: enabler
title: …
deps: [S-1]
hours: 4
refs: [AD-2]
narrative: >
  Why this enabler exists — one sentence explaining the non-user-facing need.
ac_behavioral: []
ac_nonbehavioral:
  - "…"
\`\`\`

### K-1 — <title>
\`\`\`yaml
id: K-1
type: spike
title: …
deps: []
hours: 8
refs: [AD-3]
narrative: >
  Spike question: <the specific question this spike must answer>.
ac_behavioral: []
ac_nonbehavioral:
  - "Decision recorded in plan.md under ## Outcome heading."
  - "Prototype discarded; no production code committed."
\`\`\`

**Definition of done.** Decision recorded in plan.md as an assumption note appended to
the K-item's body, under a ## Outcome heading. If the outcome requires a formal
Architectural Decision change, stop and instruct the user to run /spec-design <slug>
to revise requirements.md explicitly. The K-item's YAML narrative field is updated to
summarise the outcome.

## Additional Code Surface
Files, reusable code, tests, or conventions discovered during planning that were not
in requirements.md section 11. Same shape as the requirements.md Code Surface section.
This is the **only** place new code surface gets recorded — requirements.md stays frozen.

- **Touched files / modules** — paths + one-line rationale.
- **Reusable code** — paths + what it provides.
- **Existing tests** — paths only.
- **Patterns & conventions** — newly discovered constraints.
- **Updated starting points** — if the original top 3–5 changed materially.

If this section ends up large, it is a signal that requirements.md is stale and should
have been reopened via /spec-design — flag it as a risk.

## Open Questions carried forward
Items the user explicitly approved to carry from requirements.md, plus any new ones
surfaced during planning. Each entry: question, status (`accepted-open` |
`resolved-in-plan`), owning item ID if relevant.
```

### 9. Review loop

**Self-critique before each presentation.** Before showing the draft (and again before
each revised version), spawn **2 parallel Agent calls**: one challenging estimates as
optimistic, one challenging that the slicing is wrong (too coarse / too fine). Revise.

Present the draft. Invite edits. Iterate.

**Exit condition.** Skill ends only when the user explicitly says "approved",
"accepted", "ship it", or equivalent. Implicit silence is not approval.

On approval:
1. Update `_meta.md`: `status: plan-approved`, add `plan_approved: <today's date>`.
2. Print the exact next command:

   ```
   Approved. Next: /spec-prepare <feature-slug>
   ```

## Behaviour rules

- **Write only `.specs/<slug>/plan.md` and `.specs/<slug>/_meta.md`.** Never edit
  `requirements.md` — gaps go back through `/spec-design`.
- **No silent NFR drops.** Every NFR lands as AC on some item.
- **No silent assumptions.** Either resolve, carry forward with explicit approval, or
  reopen design.
- **Estimates are honest hours, not aspirations.** When uncertainty is high, prefer a
  spike over an optimistic estimate.
- **Never auto-run `/spec-prepare`.** Always wait for the user.
- **Item IDs are permanent once downstream files reference them.** A reopen (Step 1) may
  add or amend items but never renumbers or removes an `S-`/`E-`/`K-` ID that `pbis.md` or
  `implementation.md` already cites.
- **Reopening after `plan-approved` or later requires explicit confirmation** (Step 1) and
  must show what's downstream (existing PBIs, existing implementations) before asking.

## Tips for the user

- If you don't know a hard answer, say "spike it" — the skill will create a K-item
  rather than guess.
- Push back on slicing if a story feels too big or too thin. The skill will re-slice.
- "Challenge me harder" / "ease up" tunes the adversarial intensity, same as
  `/spec-design`.
