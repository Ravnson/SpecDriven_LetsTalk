---
name: spec-implement
description: |
  Use this skill when the user invokes /spec-implement with a feature slug, a plan item ID
  (S-1 / E-1 / K-1), or a PBI ID whose spec folder is already prepare-approved. It requires
  pbis.md — the PBI is the harness the implementation is held to, whether or not it is also
  synced to ADO. It is step 4 — the final, code-writing step — of the spec-driven chain
  (spec-design → spec-plan → spec-prepare → spec-implement) and replaces /implement-task.
  The skill is local and offline with exactly one narrow, read-only exception: it never
  pushes to a remote, never opens a pull request, never writes to or comments on Azure
  DevOps, and never searches the web — the sole exception is a single read-only fetch of
  the live acceptance criteria from the ADO work item for reconciliation. It resolves the
  target from the local .specs/<slug>/ files, refuses to run without pbis.md, cross-checks
  the PBI against plan.md, blocks on unfinished predecessor items, runs a .NET toolchain
  preflight (NuGet restore and build are allowed) and routes to the dotnet/skills plugins
  (dotnet, dotnet-test, dotnet-msbuild, dotnet-aspnetcore, dotnet-data, dotnet-blazor,
  dotnet-diag, dotnet-nuget), produces a SOLID design pass that the user approves before any
  code is written, creates a local git branch in a worktree, implements test-first in
  logical commits with 4-5 word Title Case messages (AI attribution per the repo's configured policy), enforces
  clean-code and software-craftsmanship gates via same-context self-critique, then runs
  three independent (context-free) checks before hand-off: a fresh-eyes code review with no
  visibility into the design rationale, a security review of the diff, and an independent
  AC-to-test mapping that re-derives from scratch whether each acceptance criterion is
  actually proven — reconciled against the live ADO acceptance criteria — rather than
  trusting the implementer's own claim. It verifies build/tests/format/coverage, and
  presents the diff plus the exact commands the user can run themselves to push and raise a
  PR. An item already recorded as implemented — whether named explicitly or reached via a
  feature already marked implement-done — requires explicit confirmation before any rework
  begins. Handles spikes (K-items) as an investigation-only fast path.
  Trigger phrase: "/spec-implement <feature-slug> [item-id]" or "/spec-implement <item-id>"
version: 3.2.0
argument-hint: <feature-slug | item-id | pbi-id> [item-id] [branch-name] [base-branch]
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Edit
  - Bash
  - PowerShell
  - Agent
  - Skill
  - AskUserQuestion
  - TodoWrite
  - EnterWorktree
  - ExitWorktree
  - mcp__azure-devops__wit_get_work_item
---

# spec-implement

Step 4 of the spec-driven chain, and the only step that writes production code. Takes one
approved plan item, designs it against SOLID, implements it test-first on a local branch,
proves every acceptance criterion, and hands back a diff.

**Implementation language is almost always .NET/C#.** The skill routes to the
`dotnet/skills` plugins wherever one covers the situation, and enforces the craftsmanship
rules in `references/craftsmanship.md`.

## Offline by design

This skill is **local**, with exactly **one** narrow, read-only exception. It must never:

- call any Azure DevOps tool other than the single read-only acceptance-criteria fetch in
  Step 15b — no state change, no comment, no artifact link, no PR, no other field or work
  item read;
- run `git push`, `git fetch`, `git pull`, `git remote`, or any other command that talks to
  a remote;
- create or update a pull request;
- use `WebSearch`, `WebFetch`, `curl`, or any other network access.

Only `mcp__azure-devops__wit_get_work_item` is in `allowed-tools`, and it is used exactly
once per run, in exactly one step. If any other step seems to need ADO, or the network,
the answer is to **ask the user** or **record it locally**, never to reach out.

The local `.specs/<slug>/` files are the primary source of truth. `pbis.md` is **required**
and is the implementation harness (see below); its ADO ID is used for lookup, display, and
the one reconciliation fetch below — never for anything else.

**The one exception: pulling the live AC for reconciliation (Step 15b).** The acceptance
criteria drafted into `pbis.md` can drift from what is actually on the ADO work item —
someone may have edited it there after `/spec-prepare` mirrored it. Because the whole point of
the AC verification matrix is to prove the *right* acceptance criteria, Step 15b makes a
single `GET` of `Microsoft.VSTS.Common.AcceptanceCriteria` (and Title, to confirm identity)
for this item's real ADO ID, purely to check the local copy against it. It never writes,
comments, links, or changes any state; it is skipped outright if the item has no real ADO
ID yet or the call fails, and it never blocks the run.

**Building is exempt.** `dotnet restore`, `dotnet build`, and `dotnet test` may reach
nuget.org exactly as they normally do — NuGet restore is explicitly allowed. The offline
rule is about services and remotes, not about the toolchain doing its job.

**Handing off online work.** Pushing the branch, creating the PR, and updating the work
item's state/links/comments are the user's actions. The skill ends by printing the exact
commands to do them (Step 17) — it does not run them, and it does not offer to.

## Invocation

```
/spec-implement <feature-slug> <item-id> [<branch-name>] [<base-branch>]
/spec-implement <feature-slug>
/spec-implement <item-id-or-pbi-id>
```

- `feature-slug` — the `.specs/<slug>/` folder. With no item ID, resolves to the **next
  unimplemented item** in the plan's execution order and confirms the choice.
- `item-id` — a plan item ID: `S-<n>` (story), `E-<n>` (enabler), `K-<n>` (spike).
- `pbi-id` — all digits. Looked up **locally** in the `.specs/*/pbis.md` PBI Maps. No ADO
  call is made; the ID is only a key into the local mirror.
- `branch-name` — optional. Default: `feature/<slug>-<item-id>-<kebab-title>` (title from
  the plan item, lowercased, non-alphanumerics collapsed to `-`, truncated to 60 chars).
- `base-branch` — optional. Default resolution order: `CLAUDE.md` → the current branch if
  the user confirms it → `main` / `master`, whichever exists locally. Resolve with
  `git branch --list` only; never consult the remote. Confirm before use.

Examples:
```
/spec-implement csv-export S-1
/spec-implement csv-export
/spec-implement 12345
/spec-implement csv-export S-2 feature/csv-export-writer release/2.1
```

## Chain context

1. `/spec-design` → `requirements.md`
2. `/spec-plan` → `plan.md`
3. `/spec-prepare` → ADO PBIs + `pbis.md` *(required — this skill will not run without it)*
4. **`/spec-implement`** → local branch and commits (this skill, replaces `/implement-task`)

One invocation implements **exactly one item**. The skill never chains to the next item on
its own; it ends by printing the next command.

## Reference files

Load these at the point of need, not up front:

- `references/craftsmanship.md` — SOLID rules, clean-code rules, .NET-specific rules, the
  test-quality rules, and the review checklist used in Steps 9, 11, 12 and 13. **Read it in
  full before Step 9.**
- `references/dotnet-skills.md` — the situation → dotnet/skills routing table plus the
  local availability check. **Read it in Step 6.**

## Cross-cutting rule: self-critique via parallel agents

Same rule as the rest of the chain. Wherever Claude is forming a design opinion or is
about to present something to the user, it **must** spawn **2–3 Agent calls in parallel**
with adversarial prompts attacking its own current thinking, and fold the findings back in
**before** presenting. Examples specific to this step:

- "Read this design note and name every SOLID violation and every place a test seam is
  missing."
- "Argue that this implementation does more than the item asked for."
- "Find an acceptance criterion that no test actually proves."
- "This code compiles and the tests pass. Find the behaviour it still gets wrong."

Agents inherit the offline rule: they read the local repository, they do not go online
(the sole exception, `wit_get_work_item`, is called directly by the skill in Step 15b, not
by a spawned agent). The steps below name where self-critique is mandatory.

**Two different kinds of check use the parallel-agent pattern, and they are not
interchangeable:**

- **Self-critique** (Steps 9, 12) — an agent attacks the current design or diff, but may be
  given the reasoning behind it (the design note, prior findings). Fast, and good at
  catching an obvious gap the author already half-suspects.
- **Independent review** (Steps 13, 14, 15c) — an agent receives **no design note, no
  self-critique findings, and no implementation rationale** — only the artifact itself
  (the diff, the tests, the AC text). This is what actually catches a blind spot the
  implementer's own reasoning reinforced rather than exposed. If an "independent" agent can
  see why a decision was made, it is not independent — it is another self-critique pass,
  and the step has not done what it claims to do.

## Steps

### 1. Resolve the target and check preconditions

**Resolve the spec folder and the item.**

- Argument matches `^[SEK]-\d+$` → plan item ID. Grep `.specs/*/pbis.md` PBI Maps for that
  ID. Exactly one hit is required; on multiple hits, ask which slug.
- Argument is all digits → PBI ID. Grep every `.specs/*/pbis.md` PBI Map for it and read
  the plan item ID from that row.
  - Zero hits → the ID is in no PBI Map. Tell the user, and ask for the slug and item ID
    instead. Stop.
- Argument is a slug → read `.specs/<slug>/pbis.md` and `plan.md`. Walk the ordered
  execution list and pick the first item not recorded as implemented (see below). Confirm
  before continuing.
  - **No such item exists** (every item is already implemented) → do not pick one. Report
    that `<slug>` is fully implemented and stop; a bare slug never implies rework. To
    redo a specific item, the user must name it explicitly (see the `implement-done`
    handling below).

**What counts as already implemented** — determined locally, in this order:
1. `.specs/<slug>/implementation.md` has a completed section for the item.
2. Failing that, the `pbis.md` PBI Map status column **begins with** `implemented` — this
   matches `implemented`, `implemented (local)`, and `implemented (spike)` alike. Do not
   require an exact match; Step 18 and the spike path both write a qualified value, never
   the bare word.

**Already-implemented short-circuit.** Whichever way the target item was resolved above —
named explicitly, or picked as "next unimplemented" from a slug — if it is already
implemented per the check above, **stop before doing any further work.** Ask via
`AskUserQuestion`: *"<item-id> is already recorded as implemented (branch `<branch>`,
completed `<date>`). Rework it anyway?"* Only on explicit confirmation does the run
continue; treat a confirmed rework as a reopen — proceed through Step 2 onward normally,
and set `_meta.md status: implement-in-progress` for the duration (Step 18 restores
`implement-done` once every item is implemented again).

**Check `_meta.md`.**

- Missing → `Run /spec-design <slug> first.` Stop.
- `status` is `prepare-approved` or `implement-in-progress` → allowed.
- `status` is `implement-done` → every item in the feature is already implemented. This is
  only a valid entry when the invocation names a specific item or PBI ID — there is no
  "next unimplemented item" to resolve, so a bare-slug invocation stops here with that
  explanation. A named item is allowed through; the short-circuit above then asks whether
  this is a deliberate rework.
- `status` is `plan-approved` → **stop.** The plan exists but the work has not been broken
  into PBIs. Print `Run /spec-prepare <slug> first.`
- Any other status (`design-in-progress`, `design-approved`, `plan-in-progress`,
  `prepare-in-progress`) → the pipeline hasn't reached PBI stage yet. Report the current status
  and print the command that advances it. Stop.

**`pbis.md` is required.** It is not an optional mirror — the PBI is the unit of work and
the harness this implementation is held to, independently of whether it is also synced to
Azure DevOps. Every run must resolve:

- the `pbis.md` PBI Map row for the item (plan ID, ADO ID, type, title, status), and
- the item's `PBI Details` section (description, acceptance criteria, refs, estimate).

If `pbis.md` is absent, print `Run /spec-prepare <slug> first.` and stop. If the file exists but
has no row or no Details section for this item, that is a gap in the PBI breakdown, not
something to improvise around: report the specific gap and send the user to
`/spec-prepare <slug>`. A `skipped` row is not implementable either — ask before proceeding.

**Working tree preflight.** Run `git status --porcelain`. If the tree is dirty, stop and
ask the user to commit or stash first — the skill will not create a worktree over
uncommitted work.

### 2. Load the spec chain context

Read, in this order:

1. `.specs/<slug>/pbis.md` — **the harness.** The PBI Map row (plan ID, ADO ID, type,
   title, status) and the item's PBI Details section: description, acceptance criteria,
   refs, estimate. This is what the finished work is judged against.
2. `.specs/<slug>/plan.md` — the YAML block of the item (`id`, `type`, `title`, `deps`,
   `hours`, `refs`, `narrative`, `ac_behavioral`, `ac_nonbehavioral`), the ordered
   execution list, the dependency graph, the Additional Code Surface section, and any
   carried-forward open question naming this item.
3. `.specs/<slug>/requirements.md` — the FR/NFR/AD entries named in the item's `refs`, and
   §11 Code Surface. Read the referenced requirements in full; do not work from the PBI
   description alone.
4. `.specs/<slug>/implementation.md` if it exists — prior items of this feature, their
   branches, deviations, and recorded follow-ups.

**Validation.** The PBI Details section must carry a description and at least one
acceptance criterion, and the plan item must have a complete YAML block. If either is
incomplete, report the specific gap and send the user back — `/spec-prepare <slug>` for a PBI
gap, `/spec-plan <slug>` for a plan gap. Do not improvise missing acceptance criteria.

### 3. Consistency check (the PBI is the contract)

Compare the `pbis.md` PBI Details section for this item against the `plan.md` YAML block —
title, narrative, and every AC entry. They should be identical; `/spec-prepare` derived one
from the other.

- **Consistent** → continue. Implement against the PBI.
- **Divergent** → something was edited after the breakdown. Present the differences and ask
  via `AskUserQuestion`:
  1. **PBI wins** (default) — the PBI is the agreed unit of work and the harness, so its AC
     is what gets implemented. Record the divergence in `implementation.md` and flag that
     `plan.md` should be brought back in line via `/spec-plan`.
  2. **Plan wins** — the PBI breakdown was stale; implement the `plan.md` AC and re-run
     `/spec-prepare <slug>` afterwards to correct the PBI.
  3. **Reopen** — the difference is substantive; stop and print `Run /spec-plan <slug>`
     (or `/spec-design <slug>` if a requirement itself changed).

Do **not** call Azure DevOps to break the tie. If the user needs the live work item
checked, they do that themselves and tell you the answer.

### 4. Dependency and readiness gate

From the item's YAML `deps`, check each predecessor **locally**: is it recorded as
implemented in `implementation.md`, or marked `implemented` in the `pbis.md` map?

If any predecessor is not, **block** and offer:

1. **Implement the predecessor first** — print `/spec-implement <slug> <predecessor-id>`
   and stop.
2. **It is done, just not recorded** — the user confirms; record the correction in
   `implementation.md` and continue.
3. **Proceed anyway** — requires explicit approval. Record the reason in
   `implementation.md` as a risk.
4. **Correct the graph** — the dependency is wrong; send the user to `/spec-plan <slug>`.

Also surface any **carried-forward open question** from `plan.md` that names this item. An
`accepted-open` question touching this item is an implementation risk — get a disposition
before writing code.

### 5. Spike fast path (`K-*` items only)

If the item `type` is `spike`, this is an investigation, **not** a feature. Skip Steps
9–17 and do this instead:

1. Restate the spike question from the YAML `narrative` and the time box from `hours`.
2. Investigate **locally**: read the code, run experiments, prototype in a scratch
   worktree, benchmark (`dotnet-diag: microbenchmarking`), read the repo's own docs and the
   locally installed SDK/package documentation. No web research — if the question cannot be
   answered from the repository and local tooling, say so and ask the user to supply the
   external input.
3. **Self-critique.** Spawn 2 parallel Agent calls: one arguing the conclusion is wrong,
   one naming an option that was not evaluated. Fold in.
4. Present the finding and the recommendation. Get explicit user agreement on the decision.
5. **Narrow exception to the write rules:** append an `## Outcome` section to that K-item's
   body in `plan.md` (decision, evidence, consequences for dependent items) and update the
   item's YAML `narrative` to summarise the outcome. This is the *only* edit this skill may
   make to `plan.md`.
6. **Discard the prototype.** No production code is committed from a spike. Remove the
   scratch worktree.
7. If the outcome changes an Architectural Decision, stop and print
   `Run /spec-design <slug>` to revise requirements.md before dependent items proceed.
8. Record the outcome in `implementation.md`, mark the item `implemented (spike)` in
   `pbis.md`, and print the next command. Closing the work item in ADO is the user's
   action — mention it, do not do it.

### 6. .NET toolchain preflight

**Read `references/dotnet-skills.md` now.**

Detect the stack:

```bash
git rev-parse --show-toplevel
```

Then read, when present: `global.json`, `Directory.Build.props`, `Directory.Packages.props`,
`.editorconfig`, `nuget.config`, the solution file (`*.sln` / `*.slnx`), the relevant
`*.csproj` files, and `CLAUDE.md`.

Record, and carry into the design pass:

- TFM(s), `LangVersion`, `Nullable`, `ImplicitUsings`, `TreatWarningsAsErrors`.
- Analyzer set: `EnableNETAnalyzers`, StyleCop, Roslynator, custom rulesets, `.editorconfig`
  severities. **These are the style contract — they outrank any personal preference.**
- Central package management (`ManagePackageVersionsCentrally`) — if on, package versions
  go in `Directory.Packages.props`, never in the `.csproj`.
- Test framework (MSTest / xUnit / NUnit / TUnit) and test platform (VSTest vs
  Microsoft.Testing.Platform).
- Build and test commands from `CLAUDE.md`; fall back to `dotnet build` / `dotnet test`.
- The `ai-attribution` policy from `CLAUDE.md` (`trailer` | `none` | `custom: <text>`);
  default `trailer`. See "AI attribution" below.

**New package dependencies.** Restore is free — `dotnet restore` may pull from nuget.org as
normal. Taking on a *new* dependency is still a design decision, not a build detail: name it
in the design note (Step 9) with the reason no framework type or already-referenced package
will do, and let it be approved with the rest of the design. Where central package
management is enabled, the version goes in `Directory.Packages.props`.

**dotnet/skills availability check.** Filesystem only — look for installed plugins under
`~/.claude/plugins/marketplaces/*/plugins/` and cross-check the available-skills listing in
the session context.

- **Available** → use the routing table in `references/dotnet-skills.md` throughout.
- **Not available** → mention it once, note that installing them is a separate online
  action for the user, and continue with the plain `dotnet` CLI. Do not block.

**Non-.NET repository.** If no .NET project is found, say so plainly, ask the user to
confirm the language, and apply the language-agnostic half of `references/craftsmanship.md`
(SOLID, simple design, test-first, commit hygiene). The .NET-specific gates are skipped.

### 7. Code surface, style study, and conflict check

Start from `requirements.md` §11 Code Surface and `plan.md` Additional Code Surface as
ground truth. **Do not re-discover from zero.** Then extend it for *this item only*:

- Spawn 1–2 **Explore agents** to locate the exact types and members this item touches, and
  their existing tests.
- Read the top 3–5 files in full. Note the dominant conventions: namespace style
  (file-scoped or block), field naming (`_camelCase` or otherwise), XML-doc verbosity,
  DI registration style, error-handling idiom, async idiom, test naming, test data
  builders. **New code must be indistinguishable in style from its neighbours.**
- Identify the architectural layer the change belongs in and the allowed dependency
  direction. A change that would invert the layering is a design smell — raise it in
  Step 9, do not just do it.

**Concurrent change isolation.** Scan the other `.specs/*/_meta.md` files for
`status` ≠ `implement-done` and compare their `claimed_paths` against the files this item
will touch. Report overlaps as a warning with the owning slug. Do not block — the user
decides whether to coordinate, defer, or accept.

### 8. Already-implemented check

Before designing anything, verify the acceptance criteria are not already satisfied. Grep
for the key concepts from the AC and inspect the candidate files from Step 7.

- **Fully implemented** → report precisely (file, type, member, the test that proves it).
  Do not create a branch. Record it in `implementation.md` as already done and tell the
  user the work item can be closed by hand. Stop.
- **Partially implemented** → report what exists and what is missing. Narrow the scope to
  the gap and continue.
- **Not implemented** → continue.
- **Ambiguous** → `AskUserQuestion`. Do not guess.

### 9. Design pass — SOLID by construction (approval gate)

**Read `references/craftsmanship.md` in full before drafting.**

Produce a short design note — prose and signatures, no implementation bodies:

```markdown
## Design — <item-id> <title>

### Change summary
One paragraph: what behaviour exists after this item that does not exist now.

### Types and responsibilities
| Type | New/Changed | Single responsibility (one sentence) | Layer |

### Public surface
Signatures of new/changed public and internal members, with nullability annotations.

### Dependencies and seams
Every new dependency, its abstraction, its DI lifetime, and how it is faked in tests.
Every I/O, clock, randomness, or environment touch point named explicitly.
Any new package the solution does not already reference, with the justification.

### SOLID review
SRP / OCP / LSP / ISP / DIP — one line each: how the design satisfies it, or why the
tension was resolved the way it was.

### AC to test plan
| AC | Type (unit/integration) | Test name | Test project |
Every ac_behavioral and ac_nonbehavioral entry appears exactly once. No orphans.

### Commit plan
Ordered list of independently-committable units, one line each. Each unit must build and
keep the suite green on its own.

### Explicitly out of scope
What a reader might expect but this item will not do, and why.

### Risks and follow-ups
Anything that will be recorded rather than fixed.
```

**Self-critique here is mandatory.** Spawn **3 parallel Agent calls**:

1. "Name every SOLID violation in this design and every missing test seam."
2. "Argue this design is over-engineered — which abstraction has exactly one
   implementation and no test-seam justification? Apply YAGNI."
3. "Argue this design is under-engineered — where will the next requirement force a
   rewrite instead of an extension? Which AC has no honest test?"

Reconcile 2 and 3 explicitly; do not silently pick one. Revise the note.

**Build the todo list** with `TodoWrite` from the commit plan — one todo per commit.

**Approval gate.** Present the design note and the commit plan. Proceed only on an explicit
"approved" / "go" / "looks good". Implicit silence is not approval. Iterate on request.

### 10. Create the local branch and worktree

**a. Create the worktree on a real local branch.**

```bash
safeName=$(echo "<branch-name>" | tr '/' '-')
git worktree add -b "<branch-name>" ".claude/worktrees/$safeName" "<base-branch>"
```

The branch is local only. **Do not push. Do not set an upstream. Do not touch the remote in
any way.** The user pushes when they choose to; Step 17 prints the command.

**b. Enter the worktree.** Call `EnterWorktree` with
`path: .claude/worktrees/<safeName>`.

**c. Record.** Write `_meta.md` `status: implement-in-progress`, and append the branch,
base branch, item ID, PBI ID from the PBI Map, and start date to
`.specs/<slug>/implementation.md`. Add the files named in the design note to `_meta.md`
`claimed_paths`.

Moving the work item to an in-progress state, assigning it, and linking the branch in Azure
DevOps are the user's actions. Note them in the Step 17 hand-off.

### 11. Implement test-first, in logical commits

Work the todo list one item at a time. For each unit:

1. **Red.** Write the failing test first, named per the repo convention (default
   `Method_Scenario_ExpectedResult`). Run it and confirm it fails for the right reason.
   Use `dotnet-test: run-tests` and `filter-syntax` for a tight loop.
2. **Green.** Write the simplest code that passes. No speculative generality.
3. **Refactor.** Apply `references/craftsmanship.md` to the code you just wrote —
   naming, duplication, method size, guard clauses, immutability, disposal, async
   correctness — while the suite stays green.
4. **Verify.** Build with the repo's warning settings and run the affected tests. A unit is
   not done while the build emits a new warning.
5. **Commit** (locally — never `git push`). A **4–5 word, Title Case** message, no body:
   - Good: `Add Order Validation Service`
   - Good: `Extract Export Path Resolver`
   - Bad: `feat: add validation service` (wrong style)
   - Bad: `Add the order validation service described in the item` (too long)
   - **AI attribution follows the policy in `CLAUDE.md`** (see "AI attribution" below).
     Never add promotional text or the skill's name to a commit message.
6. Mark the todo complete.

**Scope discipline.** Edit only what the unit requires. Do not fix unrelated bugs, do not
reformat untouched files, do not upgrade packages that this item does not need.

**Bounded boy-scouting.** Cleanup is allowed *only* in files this item already touches, and
only as its **own** `refactor` commit separated from behaviour change. Anything larger goes
to the Follow-ups section of `implementation.md` for the user to raise as a work item.
Never mix a refactor and a behaviour change in one commit.

**Ambiguity.** If the AC turns out to be ambiguous mid-implementation, stop and
`AskUserQuestion`. Do not guess and do not quietly widen the scope. If the answer changes
the spec, record it as a deviation in `implementation.md` and flag it for `/spec-plan`.

**Build failures.** For anything beyond a trivial compile error, use
`dotnet-msbuild: binlog-generation` + `binlog-failure-analysis` rather than guessing.

### 12. Craftsmanship review gate

With all units committed, review the **whole diff** against
`references/craftsmanship.md` — not just the last commit.

Automated checks, in this order (fix and commit until each is clean):

```bash
dotnet build -warnaserror
```

```bash
dotnet format --verify-no-changes
```

```bash
dotnet test
```

Adjust to the commands `CLAUDE.md` specifies. If `dotnet format` is not part of the repo's
workflow, rely on the analyzers and `.editorconfig` instead — do not reformat a repo that
does not ask for it.

Then run the quality skills when available: `dotnet-test: test-anti-patterns`,
`test-smell-detection`, `assertion-quality`, and `grade-tests` on the new tests.

**Self-critique here is mandatory.** Spawn **3 parallel Agent calls** over the diff:

1. "This code compiles and the tests pass. Find the behaviour it still gets wrong —
   boundary values, null, empty collection, cancellation, concurrency, culture, time zone."
2. "Grade this diff against SOLID and the Rules of Simple Design. Name the worst violation
   and the smallest change that fixes it."
3. "Find every test that would still pass if the implementation were broken, and every
   acceptance criterion that no test actually proves."

Fold the findings in as further commits. **Findings you choose not to fix must be recorded**
in `implementation.md` under Follow-ups with a one-line reason — never dropped in silence.

### 13. Fresh-eyes review (independent of implementation context)

Every check so far — the design self-critique (Step 9) and the craftsmanship self-critique
(Step 12) — runs in the same context as the one that wrote the code, which is fast but has
a blind spot: it tends to re-confirm the reasoning that produced the code in the first
place. This step is different in kind, not just in prompt.

Spawn a **fresh Agent with no context from this conversation** — it must not receive the
design note, the self-critique findings, or any commit-by-commit rationale. Give it only:

- `git diff <base-branch>...<branch-name>` — the full diff, not a summary;
- the PBI Details section (description + acceptance criteria) from `pbis.md`;
- the relevant `CLAUDE.md` conventions.

Prompt it to review as a colleague seeing this change for the first time in a pull request,
with no access to the reasoning behind it: correctness, readability, naming, whether the
diff does what the PBI says and nothing more, and whether a reviewer unfamiliar with the
discussion could understand every change from the diff alone.

Run **2 such agents in parallel** with different framings for coverage — e.g. one on
"correctness and edge cases," one on "reuse, naming, and simplification" — rather than one
generic pass.

Triage findings like Step 12's: fix with a further commit, or record in
`implementation.md` under Follow-ups with a one-line reason. A finding that shows the diff
does not actually satisfy the PBI is never a follow-up — fix it before continuing.

### 14. Security review

Spawn a **fresh Agent, same isolation as Step 13** (no design note, no prior findings),
scoped to security, reviewing the diff. At minimum check for:

- **Injection** — SQL, command, LDAP, XPath, log injection from any new external input.
- **Unsafe deserialization** — untrusted data deserialized, unsafe reflection, unchecked
  casts driven by input.
- **AuthN/AuthZ** — every new endpoint or handler enforces the access control the rest of
  the codebase enforces at that layer; no bypassed, weakened, or missing check.
- **Secrets** — no credential, connection string, key, or token added in code, test
  fixtures, or config (including `appsettings.*.json`).
- **Path traversal / unsafe file or archive access** from user-controlled input.
- **SSRF** — any new outbound call built from user-controlled input.
- **Weak cryptography** — custom crypto, weak algorithms/modes, hardcoded IVs/keys,
  insecure randomness for security-sensitive values.
- **Missing input validation** at any trust boundary the change introduces or touches.
- **Sensitive data in logs** — PII, secrets, or tokens logged at any level.

This covers the same ground as the `security-review` skill, run locally against the branch
diff — no new network access beyond what Step 6 already covers.

Findings are severity-ranked. **Any finding above low severity blocks the hand-off** — fix
it before Step 15. Low-severity findings are recorded in `implementation.md` under
Follow-ups with the reason they were deferred, same as any other finding.

### 15. Acceptance criteria verification matrix

This is the highest-value check in the gate: it is where "the tests pass" gets turned into
"the acceptance criteria are actually proven." Three parts, in order — do not skip (b) or
(c) even when (a) looks clean.

**a. Draft the matrix locally.**

| AC (from the PBI) | Kind | Proven by | File | Result |
|---|---|---|---|---|
| Given … When … Then … | behavioral | `Export_WithEmptyOrderSet_WritesHeaderOnly` | `tests/…/ExporterTests.cs` | pass |
| p95 < 500 ms at 1k users (NFR-1) | non-behavioral | `ExportBenchmark` | `bench/…` | pass |

Rules:
- **Every** `ac_behavioral` and `ac_nonbehavioral` entry from the PBI appears.
- Default expectation is an **automated** test. A manual verification is allowed only with
  explicit user approval, recorded as `manual (approved <date>)`.
- A non-behavioral AC that cannot be automated (e.g. a documentation requirement) must
  still name concrete evidence — the file and section that satisfies it.

**b. Pull the live AC from Azure DevOps — the one exception to the offline rule.**

If the PBI Map has a real ADO ID for this item (not `—`, not a draft placeholder), call
`mcp__azure-devops__wit_get_work_item` **once**, read-only, for that ID, and read only
`Microsoft.VSTS.Common.AcceptanceCriteria` (and Title, to confirm identity). This is the
*only* Azure DevOps call this skill ever makes. It never writes, comments, links, or
changes state.

- **Matches `pbis.md`** → mark "ADO-verified" on the matrix and continue.
- **Diverges** → someone edited the work item directly in ADO after `/spec-prepare` mirrored it
  (or after this session started). **Do not silently prefer either version.** Present the
  diff via `AskUserQuestion`:
  1. **Implement the ADO version** — it's the one the rest of the team sees; update the
     local matrix and record the drift in `implementation.md`.
  2. **Keep the local version** — the ADO edit was wrong or premature; record that the user
     was told and chose to proceed, so a later `/spec-prepare` run doesn't silently overwrite
     the decision.
  3. **Reopen planning** — the ADO edit reveals a real scope change; stop and print
     `Run /spec-plan <slug>`.
- **No real ADO ID yet** (draft, never uploaded) → skip this sub-step, note "not synced to
  ADO" on the matrix, do not block.
- **Call fails** (auth, network, not found) → report the failure once, skip reconciliation,
  continue. This check is a cross-check, not a hard dependency — it never blocks the run by
  itself.

**c. Independent AC-to-test mapping — a fresh agent re-derives it; it does not review the
implementer's claim.**

The matrix in (a) is self-attested: the same context that wrote the code also decided which
test "proves" which AC. Do not trust that mapping. Spawn a **fresh Agent with no visibility
into the drafted matrix, the design note, or any prior finding** — give it only:

- the PBI's description and narrative, and the final AC text from (a)/(b) — the same PBI
  content Step 13's fresh-eyes agent receives. Withholding the description isn't part of
  what "independent" means here (that rule excludes the design note, the self-critique
  findings, and the implementation rationale — not the requirement's own text), and an
  agent judging AC-to-test correctness needs the domain context to do it well;
- the full test diff (new and changed test files),
- the production diff, for reference only.

Instruct it to build its **own** AC → test mapping from scratch: for each AC, find which
test(s) actually exercise it, read what they assert, and independently judge whether the
test would fail if the behaviour the AC describes were broken. It returns, per AC, one of:

- **PROVEN** — a test exists and would fail if the behaviour regressed.
- **WEAK** — a test touches the area but wouldn't catch a real regression (e.g. it checks
  only the happy path of a Given/When/Then that also specifies an error path).
- **NOT PROVEN** — no test found.

Reconcile against (a):
- Agreement on PROVEN → done.
- Independent verdict is WEAK or NOT PROVEN where (a) claimed pass → **this blocks the
  hand-off.** Add or strengthen the test; do not argue the matrix into agreeing with itself.
- Independent agent finds a test proving an AC that (a) never mapped → fold it in as a
  correction.

Record the independent agent's verdicts in `implementation.md` alongside the matrix — not
just the final reconciled state — so a reader can see what the fresh check actually said,
not only what it was corrected to.

**Blocking rule.** Any AC without a PROVEN verdict after (a)–(c) blocks Step 17. Report it
and ask how to proceed; never hand off with an unproven acceptance criterion.

### 16. Full build, test, and coverage

```bash
dotnet build
```

```bash
dotnet test
```

Run the test categories `CLAUDE.md` names for the affected area. Fix every failure before
continuing — a red suite is never handed over.

Coverage: run `dotnet-test: coverage-analysis` (and `test-gap-analysis` /
`find-untested-sources` if the repo tracks coverage) scoped to the changed files. Report the
delta for the touched code. If coverage of new code drops below the repo's threshold, add
tests rather than lowering the bar. If the repo has no threshold, report the number without
inventing a gate.

**Base branch drift.** Compare against the **local** base branch only
(`git log --oneline <base-branch>..HEAD` and `git merge-base`). If the local base moved
during implementation, rebase onto it and re-run the suite. Never `git fetch` to check
whether the remote moved — say in the hand-off that the user should rebase after fetching.

### 17. Present the diff and hand off

Show a summary of `git diff <base-branch>...<branch-name>`:

- Each changed file with a one-line rationale.
- The commit list with messages.
- The AC verification matrix from Step 15, including the ADO reconciliation status and the
  independent agent's verdicts.
- Fresh-eyes and security review findings (fixed and deferred).
- Follow-ups and deviations.

Then print the hand-off block — **commands for the user to run, not actions to take**:

```
The branch <branch-name> exists locally only. Nothing was pushed, and the only Azure
DevOps contact this run made was a single read-only fetch of the acceptance criteria
(Step 15b) — no state, comment, or link was changed. When you are ready:

  git -C .claude/worktrees/<safeName> fetch origin
  git -C .claude/worktrees/<safeName> rebase origin/<base-branch>   # if it moved
  git -C .claude/worktrees/<safeName> push -u origin <branch-name>

Then, by hand in Azure DevOps:
  - link work item <ado-id or "n/a"> to branch <branch-name>
  - move it to the in-progress / review state
  - open the PR (suggested title: "<item title>")
  - raise the recorded follow-ups as new work items
```

Include a suggested PR description in a fenced block the user can copy: the narrative, the
AC verification matrix, the `refs` traceability line, and a pointer to `.specs/<slug>/`.
Add the AI disclosure line if the `ai-attribution` policy calls for one.

**Do not offer to push or to create the PR**, even if asked to "just finish it" — say
plainly that this skill only writes locally and reads ADO read-only, and hand over the
commands. Running them is the user's call, in their own shell.

### 18. Finalise records and print the next command

1. Append the completed section to `.specs/<slug>/implementation.md`:

```markdown
## <item-id> — <title>  (PBI <ado-id or "n/a">)

- **Branch:** <branch> (base: <base>) — local only, not pushed
- **Commits:** <list of messages>
- **Files touched:** <paths>
- **AC verification:** <matrix, with ADO reconciliation status and independent-agent verdicts>
- **Fresh-eyes review findings:** <list, or "none">
- **Security review findings:** <list, or "none">
- **Deviations from spec:** <list, or "none">
- **Follow-ups:** <list with reasons, or "none">
- **Pending user actions:** push, work-item link/state, PR
- **Completed:** <date>
```

2. Update the `pbis.md` PBI Map status column for this item to `implemented (local)`.
3. Update `_meta.md`: keep `status: implement-in-progress` while any item in the plan's
   execution list is unimplemented; set `status: implement-done` and
   `implement_done: <date>` once **all** items are implemented or skipped.
4. Print the next command:

```
Done: <item-id> implemented on <branch> (local).
Next: /spec-implement <slug> <next-item-id>
```

   or, when the feature is complete:

```
All items for <slug> implemented locally. Feature complete — push and raise PRs when ready.
```

## AI attribution

Whether commits and PR text disclose AI assistance is the repository owner's decision, not
the skill's. Read the `ai-attribution` key from `CLAUDE.md` (Step 6):

- `trailer` (**default when unset**) — end each commit message with a `Co-Authored-By:`
  trailer naming the assistant, and add a one-line disclosure to the suggested PR
  description. This is the one exception to the "no body" rule for commits.
- `none` — no attribution in commit messages or PR text. Use only where the repo, team, or
  employer policy permits it.
- `custom: <text>` — use the given trailer or disclosure line verbatim.

If the repo's contribution guidelines require AI disclosure, `trailer` (or `custom`) is
mandatory regardless of `CLAUDE.md`; say so rather than silently omitting it.

## Files this skill may write

- `.specs/<slug>/implementation.md` — created and appended by this skill.
- `.specs/<slug>/pbis.md` — **status column only.** PBI titles, descriptions, and
  acceptance criteria are never rewritten here; they are the contract.
- `.specs/<slug>/_meta.md` — `status`, `claimed_paths`, `implement_done`.
- `.specs/<slug>/plan.md` — **only** the `## Outcome` section and `narrative` field of a
  K-item, on the spike fast path (Step 5).
- Source and test files in the repository, on the feature branch only.

`requirements.md` is never edited. Gaps go back through `/spec-design`.

## Behaviour rules

- **Local only, with one narrow exception.** No `git push` / `fetch` / `pull`, no PR, no web
  access — ever, and not on request. The only Azure DevOps contact allowed is the single
  read-only `wit_get_work_item` fetch in Step 15b; no other ADO tool, and no write of any
  kind. Print the commands for everything else instead of running them.
- **Independent checks stay independent.** Steps 13, 14, and 15c never receive the design
  note, the self-critique findings, or the locally-drafted AC matrix — only the artifact
  under review. An agent given the rationale is not doing an independent check.
- **Nothing ships with an open independent finding.** Any fresh-eyes finding that shows the
  diff doesn't satisfy the PBI, any security finding above low severity, and any AC without
  a PROVEN verdict from Step 15c all block Step 17 — fix them, don't argue past them.
- **One item per invocation.** Never auto-continue to the next.
- **Never silently rework an already-implemented item.** Whether picked up via
  `implement-done` or named explicitly, an item already recorded as implemented requires
  explicit confirmation (Step 1) before any further work starts.
- **No code before the design gate.** Step 9 approval is mandatory (spikes excepted).
- **The spec is the scope.** Not more, not less. Anything extra is a follow-up, not a
  commit.
- **Test-first, always.** A commit that adds behaviour without a test that would have
  failed before it is not acceptable.
- **Every commit builds and the suite stays green.** No "fix build" commits.
- **AI attribution is explicit, never improvised.** Follow the `ai-attribution` policy
  (see "AI attribution") for commit messages and the suggested PR text.
- **Analyzers and `.editorconfig` outrank preference.** Match the neighbouring code
  exactly.
- **All identifiers, comments, JSON keys, log messages, and schema fields are in English**,
  even when the domain, the spec, or the user's prompt is in another language.
- **Route to dotnet/skills** whenever the routing table covers the situation, instead of
  improvising.
- **`pbis.md` is required and is the contract.** No PBI, no implementation — the run stops
  and sends the user to `/spec-prepare`. A new package reference is justified in the design
  note, not slipped in during implementation.
- **Restore and build may use the network.** NuGet restore is explicitly allowed; the
  offline rule covers services and remotes, not the toolchain.
- **No silent drops.** Every unfixed finding, deviation, unproven AC, and pending user
  action is recorded in `implementation.md`.
- **Never force-push** (the skill does not push at all), never `git reset --hard` over user
  work, never commit secrets or `appsettings.*.json` files containing real values.

## Tips for the user

- `/spec-implement <slug>` picks up the next unimplemented item; add an item ID to choose.
- Run `/spec-prepare <slug>` first. The PBI is the harness the implementation is measured
  against, whether or not you also sync it to ADO — without `pbis.md` this skill stops.
- Everything stays on your machine except one read-only AC lookup. When you want it pushed
  or a PR opened, copy the hand-off commands from Step 17 — the skill will not do it for you.
- If the fresh-eyes, security, or independent AC-mapping check finds something, that's the
  point of running them separately from the same-context self-critique — expect them to
  occasionally disagree with the implementation's own assessment.
- Say "challenge me harder" / "ease up" to tune the adversarial intensity, same as the rest
  of the chain.
- Push back at the design gate — it is far cheaper to re-slice the design there than to
  review the diff.
- If the item turns out to be wrong rather than just hard, say so: the skill will stop and
  send you back to `/spec-plan` or `/spec-design` instead of coding around it.
