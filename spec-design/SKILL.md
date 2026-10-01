---
name: spec-design
description: |
  Use this skill when the user invokes /spec-design with a feature slug and a free-form
  description (optionally with screenshots, links, or referenced files). It is the first
  step of a spec-driven development chain (spec-design → spec-plan → spec-prepare →
  spec-implement) that replaces /implement-task. The skill performs parallel research
  (codebase, local docs, web, Azure DevOps wiki and work items), challenges the user's
  input for logic flaws and missing edges, runs batched clarification rounds via
  AskUserQuestion, and writes a rigorous requirements.md to .specs/<slug>/. Re-invoking on
  an already-approved slug is treated as an explicit reopen that shows what plan/PBI/
  implementation work already exists downstream before letting the user resume editing or
  restart. It only ends when the user explicitly approves the document.
  Trigger phrase: "/spec-design <feature-slug> <description>"
version: 1.3.0
argument-hint: <feature-slug> <description>
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
  - mcp__azure-devops__wiki_get_page_content
  - mcp__azure-devops__wit_get_work_item
---

# spec-design

First step of the spec-driven chain. Turn a natural-language idea into a rigorous
`requirements.md` through deep, challenging conversation. **No code is written.** Only
`.specs/<slug>/requirements.md` and `.specs/<slug>/_meta.md` may be created or edited.

## Invocation

```
/spec-design <feature-slug> <description>
```

- `feature-slug`: short kebab-case name — becomes the folder under `.specs/`.
- `description`: free-form text. May span multiple lines and reference screenshots, links,
  files, or related ADO items.

Example: `/spec-design csv-export "Users should be able to export the currently filtered table to CSV from the toolbar"`

## Chain context

This skill is step 1 of 4:

1. **`/spec-design`** → `.specs/<slug>/requirements.md` (this skill)
2. `/spec-plan <slug>` → `.specs/<slug>/plan.md` (user stories, acceptance criteria)
3. `/spec-prepare <slug>` → ADO PBIs + `.specs/<slug>/pbis.md`
4. `/spec-implement <pbi-id>` → branch, commits, PR (replaces `/implement-task`)

Each step is invoked explicitly by the user. This skill **never** auto-runs the next one;
it ends by printing the exact next command.

## Cross-cutting rule: self-critique via parallel agents

At every step below where Claude is forming an opinion, drafting content, or about to
present something to the user, Claude **must** spawn **2–3 Agent calls in parallel** with
adversarial prompts that attack its own current thinking. Examples:

- "Read this draft requirements.md and list every logic flaw, contradiction, and
  unstated assumption you can find. Be ruthless."
- "Argue the *opposite* architectural choice from the one currently in AD-1. What does
  it buy us? Why might it be the better call?"
- "List edge cases this spec ignores: empty state, concurrency, permission denial,
  offline, malformed input, scale, i18n, time zones, partial failure."

Critique findings are folded back into the work — the draft is revised, or the issue is
surfaced as direct pushback to the user — **before** presenting anything. Self-critique
is a first-class step, not optional polish. Each step below names where to invoke it.

## Steps

### 1. Sanity-check the user's opening message

Before any research, inspect the description for missing essentials. If any of the
following is clearly absent, send one short message asking for it, then continue:

- **Purpose, not solution.** If the description reads as a solution ("add a button that
  does X"), ask for the underlying purpose ("what should the user accomplish?").
- **Hidden constraints.** If no deadline, team size, stack lock-in, regulatory or
  compatibility constraint is mentioned, ask for any that apply.
- **Out-of-scope.** If nothing is marked as explicitly out of scope, ask what's out.

One reminder message total — not a questionnaire. Then proceed to Step 2 even if the user
defers; unanswered items become Open Questions in the final document.

### 2. Create the feature folder

```
.specs/<feature-slug>/
  requirements.md   (created in Step 6)
  _meta.md          (created now)
```

**Duplicate slug check.** Before writing anything, check whether `.specs/<feature-slug>/`
already exists. If it does, read the existing `_meta.md`, show the user the current
status and created date.

- **If `status: design-in-progress`** — an unfinished draft. Ask:
  - **Resume** (default) — continue from the current draft; skip to Step 7 with the
    existing `requirements.md` loaded for review.
  - **Restart** — archive the existing folder to `.specs/<feature-slug>-archived-<date>/`
    and begin fresh.
- **If `status` is `design-approved` or later** (`plan-approved`, `prepare-approved`,
  `implement-in-progress`, `implement-done`) — this is a **reopen** of an already-approved
  spec, and downstream artifacts likely exist. Before asking anything, read `plan.md` /
  `pbis.md` / `implementation.md` if present and show what would go stale (plan items,
  PBIs already in ADO, items already implemented). Then ask:
  - **Reopen** (explicit) — skip to Step 7 with the existing `requirements.md` loaded for
    review; on re-approval, `status` returns to `design-approved` and Step 7 appends a new
    Document History row as it already does for a re-opened design.
  - **Restart** — archive the **entire** folder — including `plan.md`, `pbis.md`, and
    `implementation.md` — to `.specs/<feature-slug>-archived-<date>/`. Make sure the user
    understands this takes the downstream artifacts with it, not just the requirements.
  - **Cancel** — leave everything as is.

If the folder does not exist, create it and write `_meta.md`:

```yaml
slug: <feature-slug>
status: design-in-progress
created: <today's date>
approver: <user email if known from session context>
claimed_paths: []    # populated from §11 Code Surface on approval
ado-project: ~       # populated when discovered in Step 3
ado-area-path: ~     # populated when discovered in Step 3
```

### 3. Parallel ingestion

Launch the following **in parallel** (single message, multiple tool calls):

1. **Explore agent — codebase.** Find existing functions, utilities, components, and
   patterns relevant to the request. Flag reusable code and likely conflict points.
   Start from entry-point directories named in `CLAUDE.md` (or repo root if absent).
   Limit to source and test file types. If a token/time limit is hit before completion,
   emit a warning in §11 Code Surface: "codebase scan incomplete — results may be partial."
2. **Explore agent — local docs.** Read `README*`, `CLAUDE.md`, `docs/**`, architecture
   notes. Surface relevant constraints and conventions.
   **CLAUDE.md fallback:** If `CLAUDE.md` is not found, record `no-claude-md: true` in
   `_meta.md`, emit a one-line warning to the user ("No CLAUDE.md found — codebase
   conventions and ADO context will need to be provided manually"), and continue with
   reduced context.
3. **WebSearch.** Prior art, libraries, RFCs, known pitfalls for the problem domain.
   Skip only if BOTH: (a) the description explicitly contains "internal only", AND
   (b) the codebase agent already found a directly reusable pattern. Otherwise default
   to running it.
4. **ADO search.** `mcp__azure-devops__search_wiki` and
   `mcp__azure-devops__search_workitem` using keywords from the description. Fetch
   promising wiki pages with `wiki_get_page_content` and related work items with
   `wit_get_work_item`.
   **ADO unavailability fallback:** If ADO tools are unavailable, return auth errors, or
   return zero results, note this in `§10 References` as "ADO search unavailable/empty"
   and continue without blocking. If zero results are returned, ask the user once:
   "Do you have a related work-item ID I should read?" — optional, no block if declined.
   If the ADO project and area path are discovered here (e.g. from a linked work item),
   write them into `_meta.md` as `ado-project` and `ado-area-path` for downstream skills
   to inherit.

**Self-critique here.** Before summarizing, spawn 3 parallel Agent calls: one to
challenge whether the research scope was sufficient ("what likely-relevant source did I
miss?"), one to argue the description means something different from what I assumed,
and one specifically to challenge the **Code Surface draft**: "what files/modules/tests
are likely touched but missing from this list? what reusable utilities did the search
miss?" Fold all findings into the summary and into the eventual Code Surface section.

Summarize findings to the user briefly before Step 4. The Code Surface inventory is
part of that summary.

### 4. Batched clarification rounds

Use `AskUserQuestion` with 3–4 grouped questions per round. Typical progression:

- **Round 1** — Purpose, users, success criteria, hard constraints, explicit out-of-scope.
- **Round 2** — Functional behavior edges, failure modes, data shape, integrations.
- **Round 3** — Non-functional: performance targets, security/auth, accessibility,
  compliance, observability.
- **Round 4** — Architectural decisions and alternatives surfaced by Step 3 research.

Rules:
- A "don't know yet" answer is valid — record it as an Open Question.
- Stop asking when further rounds would produce diminishing returns. Three rounds is
  typical; four is the cap unless the user explicitly wants more.

### 5. Challenge pass

**Self-critique here is mandatory.** Spawn **3 parallel Agent calls** with these
prompts (each agent gets the user's answers and the Step 3 research summary):

1. "Find every logic flaw, contradiction, and unstated assumption in these answers."
2. "Argue the opposite architectural direction. Make the strongest possible case."
3. "List edge cases ignored: empty state, concurrency, permission denial, offline,
   malformed input, scale, i18n, time zones, partial failure, retry semantics."

Merge their findings with your own. Then hunt explicitly for:

- **Logic flaws and contradictions** between answers.
- **Hidden assumptions** the user hasn't stated.
- **Missing edge cases**: empty state, concurrent edits, permission denial, offline,
  very large or malformed input, partial failure, retry semantics, i18n, time zones.
- **Over-scoping** vs. the stated purpose — anything that doesn't earn its place.
- **Conflicts with existing code patterns** surfaced in Step 3.

Raise these as direct pushback in chat. Do not bury them silently in the document. The
default stance is hard challenge — the user explicitly asked for deep analytical thinking.
If the user says "ease up", lower the intensity but still record issues as Risks.

### 6. Draft `requirements.md`

Write `.specs/<feature-slug>/requirements.md` with these sections, in order:

```markdown
# <Feature Name>

## 0. Document History
| Version | Date | Author | Summary |
|---------|------|--------|---------|
| 1.0 | <approval date> | <approver> | Initial approved version |

## 1. Context
Problem statement, who it's for, what pain it removes, why now.

## 2. Goals
Bulleted, specific, measurable where possible.

## 3. Non-Goals
Explicit out-of-scope items.

## 4. Functional Requirements
Numbered, testable statements. Each FR carries a mandatory RFC 2119 priority qualifier:
- `[SHALL]` — required for the feature to be viable; blocks release if missing.
- `[SHOULD]` — strongly desired; included unless constrained by time or risk.
- `[MAY]` — nice-to-have; first to be deferred if scope must be cut.

Examples:
- **FR-1** `[SHALL]` The system shall …
- **FR-2** `[SHOULD]` The system shall …
- **FR-3** `[MAY]` The system shall …

## 5. Non-Functional Requirements
Each NFR also carries a `[SHALL]`/`[SHOULD]`/`[MAY]` qualifier.
- **NFR-1** `[SHALL]` Performance: …
- **NFR-2** `[SHALL]` Security: …
- **NFR-3** `[SHOULD]` Accessibility: …
- **NFR-4** `[SHOULD]` Compliance / privacy: …
- **NFR-5** `[MAY]` Observability: …
(Omit categories that genuinely don't apply; don't pad.)

## 6. Architectural Decisions
One lightweight ADR per decision:
### AD-1: <decision title>
- **Decision:** what
- **Rationale:** why
- **Alternatives considered:** what else, why rejected
- **Consequences:** what this commits us to

## 7. Risks
Known risks with brief mitigation notes.

## 8. Open Questions
Items unresolved at end of design. Carried into /spec-plan.

## 9. Glossary
Shared domain vocabulary.

## 10. References
Links to ADO wiki pages, web sources, related work items, screenshots/files.

## 11. Code Surface
The ground-truth map of where this feature lives in the codebase. Populated from the
Step 3 codebase Explore agent. This section is the **hand-off to /spec-plan and
/spec-implement** — they read it instead of re-discovering from zero.

- **Touched files / modules** — paths likely to change, one-line rationale each.
- **Reusable code** — existing functions, classes, utilities with paths; anti-duplication anchor.
- **Existing tests** — test files that already cover related behavior; paths only.
- **Patterns & conventions** — style/layering/naming rules discovered in surrounding code that new work must respect.
- **Starting points** — ordered list of the top 3–5 files a planner should read first.

Each entry uses this machine-readable schema:
- `path` — repo-root-relative path (e.g. `src/exports/CsvExporter.cs`)
- `reason` — one-line rationale
- `confidence` — `high` | `medium` | `low`
- `greenfield` — `true` | `false`

**Greenfield rule.** If no relevant existing code exists, this section is still
**required** — fill it with the **planned structure** (modules/files the feature will
create, tests that will accompany them, conventions to establish). This is **declarative
intent only** — names and purposes, no files are created. Each entry has `greenfield: true`.
Path values are the intended future paths, not existing ones. If a greenfield §11 entry
implies a structural decision, that decision belongs in §6 as an AD; §11 references the
AD by ID rather than repeating the rationale.
```

Every FR and NFR must be testable. Vague items ("should be fast") are rewritten with a
concrete target or moved to Open Questions.

### 7. Review loop

**Self-critique before each presentation.** Before showing the draft (and again before
each revised version), spawn **2 parallel Agent calls**: one to find weaknesses in the
draft as written, one to argue a specific FR/NFR/AD is wrong or unnecessary. Revise the
draft before showing it.

Present the draft inline (or summarize and point at the file). Invite edits. Iterate.

**Exit condition.** This skill ends only when the user explicitly says one of:
"approved", "accepted", "ship it", "looks good", "done", "go ahead".
Implicit silence is not approval.

**Maximum revision count.** After 5 revision rounds, surface all remaining unresolved
disagreements in a single summary and require the user to pick one of:
- **Force-approve as-is** — proceed with open issues recorded as Risks.
- **Abandon** — archive the folder to `.specs/<slug>-abandoned-<date>/` and stop.
- **Extend** — continue for 3 more rounds (requires explicit "extend" response).

**Resume protocol.** If `/spec-design <slug>` is invoked and `_meta.md` shows
`status: design-in-progress`, show the existing draft and ask:
- **Resume** (default) — load the draft and continue the review loop from Step 7.
- **Restart** — archive and begin fresh.

On approval:
1. Update `_meta.md`: `status: design-approved`, `approved: <today's date>`,
   `approver: <user email if known>`, `revision_count: <n>`.
2. Append a row to `requirements.md §0 Document History`:
   - First approval: `| 1.0 | <date> | <approver> | Initial approved version |`
   - Subsequent approvals (re-opened design): `| 1.<n> | <date> | <approver> | <one-line summary of what changed> |`
3. Update `_meta.md` `claimed_paths` list from §11 Code Surface `path` fields —
   used by other active specs to detect concurrent file ownership conflicts.
4. Print the exact next command:

   ```
   Approved. Next: /spec-plan <feature-slug>
   ```

## Behaviour rules

- **Write only `.specs/<slug>/requirements.md` and `.specs/<slug>/_meta.md`.** No other
  edits, no scaffolding, no example code files. If editing other files would clearly help,
  surface the suggestion in the Risks or Open Questions section instead.
- **Challenge by default.** Push back on assumptions, name what's missing, refuse to
  capture vague requirements verbatim.
- **Don't guess on behalf of the user.** Use `AskUserQuestion` or record an Open Question.
- **Never auto-run `/spec-plan`.** Always wait for the user to invoke it.

## Tips for the user (and what the skill reminds you of)

The skill actively reminds you of these at the start of each run if your opening
description violates them:

- Lead with **purpose**, not solution: "users should be able to X so that Y".
- State **constraints I can't see**: deadlines, team size, stack lock-ins, regulations.
- Be explicit about **out-of-scope** items up front.
- Say "**challenge me harder**" or "**ease up**" to tune pushback.
- "**I don't know**" is a valid answer — it becomes an Open Question, not a guess.
