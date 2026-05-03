---
name: develop
description: "End-to-end development workflow with sub-agent specialization. Use when implementing features, fixing bugs, or making code changes that should follow project standards. Builds work as a stack of jj changes; lands each as its own PR. Dispatches focused sub-agents for planning, implementation, quality checks, and architectural review — keeping the coordinator lean."
---

# /develop — Sub-Agent Development Workflow

Implement changes using specialized sub-agents, each with a dedicated context window.
The coordinator (you) stays lean — orchestrate, don't accumulate.

This skill is jj-native. New work creates new commits via `jj new` rather than amending.
Logical units of work are stacked as a chain of changes; each lands as its own PR.
See [references/stacking-conventions.md](../references/stacking-conventions.md) for the
bookmark naming, push, and PR-base-fixup mechanics.

Use memory-mcp to load `required-environment-variables` and `rust-code-standards` memories
(scope: global) if not already loaded this session. Check for project-scoped memories (use
`list` filtered by project scope) — pass their contents to sub-agents as context.

## Argument handling

`$ARGUMENTS` describes the work to do. It can be:

| Form | Meaning |
|------|---------|
| Free text | Feature/bug description — run full workflow |
| `plan <text>` | Phase 1 only — produce a plan, stop |
| `implement <text>` | Phases 1-2 — plan and implement |
| `quality` | Phase 3 only — check quality of current stack |
| `review` | Phase 4 only — code review of current stack |
| `land` | Phase 5 only — fetch, rebase, push, and open PRs for the current stack |
| `issue <N>` | Fetch issue details, then run full workflow |

## Coordinator responsibilities

You are the orchestrator. Your job:
1. Parse the task and determine which phases to run
2. Gather minimal context (project language, build commands, relevant file paths)
3. Dispatch sub-agents with focused prompts and necessary context
4. Synthesize sub-agent results — resolve conflicts, surface decisions
5. Present a clear summary to the user at each milestone

You do NOT: read implementation files into your own context, write code directly,
or run tests yourself. Sub-agents do the focused work.

## Phase 0: Frame the work

Before planning begins, establish the "so what?" — why does this work matter?

- **Who benefits** from this change?
- **What's the counterfactual** — what happens if we don't do it?
- **What does success look like** and how would we know?

For small, well-scoped tasks (bug fix with a clear issue, config change), this can be a
one-sentence acknowledgment. For larger work — new features, architectural changes, greenfield
components — this is a deliberate pause to align on intent before investing in a plan.

State the framing to the user. If the "so what?" isn't clear from the task description, ask.
This framing anchors everything downstream — the plan, the implementation decisions, and the
flight log entry at the end.

### Stack starting point

Before Phase 1, ensure the working copy is positioned correctly:

```bash
jj log -r 'stack()'
```

If `stack()` is non-empty, ask whether this work belongs:
- **On top of the existing stack** (new layer building on what's there)
- **As a new stack** (the existing work needs to be pushed/landed first, or set aside)

If the user wants a fresh stack, run `jj new bookmark_base()` to start from the last
published bookmark. Don't proceed until the working position is intentional.

## Phase 1: Plan

Dispatch a **planning sub-agent** (model: opus 4.6) to:

1. If `$ARGUMENTS` references an issue, fetch it: `gh issue view <N> --json title,body,comments`
2. Read the relevant code using Serena's symbolic tools — `get_symbols_overview` for structure,
   `find_symbol` with `include_body=true` only for symbols that need modification
3. Identify all files and symbols that need to change
4. For each changed function/method signature, use `find_referencing_symbols` to find callers
5. For stateful subsystems: identify resource lifecycle (creation → cleanup → limits).
   External connections/sessions require a timeout and max-count strategy in the plan.
6. **Decompose into stack layers.** If the work is large enough to warrant a stack, propose
   the layer breakdown — what each layer contains, why the order, where the seams are.
   Each layer should be:
   - Reviewable on its own (passes tests, makes sense in isolation)
   - Cohesive (one logical concern)
   - Roughly 100-300 lines of diff (the sweet spot for review quality)
7. Propose an approach: what changes, in what order, and why
8. Flag risks, ambiguities, or decisions that need user input

**Sub-agent prompt template:**
```
You are a planning agent. Your job is to understand the task and propose a concrete
implementation approach. Do NOT write code — produce a plan.

Task: <task description>
Project language: <language>
Build command: <from CLAUDE.md or Cargo.toml etc.>
Project conventions: <from memory-mcp project memories if available>
Current stack: <output of jj log -r 'stack()' if non-empty, else "empty">

Use Serena's symbolic tools to explore the codebase efficiently:
- get_symbols_overview for file structure
- find_symbol with include_body=true only for symbols you need to understand deeply
- find_referencing_symbols for impact analysis

Decide whether the work fits in a single layer or warrants a stack:
- Single layer: small, cohesive, <300 lines of expected diff
- Stack: multiple cohesive layers; each independently reviewable

Output format:

## Plan

### Layer 1: <intent>
1. [Change description] — `file:symbol`
   - Why: [rationale]
   - Impact: [callers/dependents affected]
2. ...

### Layer 2: <intent>  (if stacked)
...

## Risks
- [risk description and mitigation]

## Questions (if any)
- [question for the user]
```

Present the plan to the user. **Always wait for explicit approval before proceeding
to Phase 1.5.** Do not auto-proceed — the user reviews and greenlights every plan.

## Phase 1.5: Record decisions (ADRs)

After the plan is approved, write Architecture Decision Records for any significant
decisions made during planning. This is the coordinator's job — no sub-agent needed.

ADRs live in `docs/adr/` in the project repo. Use sequential numbering:
`0001-short-title.md`, `0002-short-title.md`, etc. Check existing ADRs to get the
next number.

**Format:**
```markdown
# ADR-NNNN: <Title>

## Status
Accepted

## Context
<Why this decision was needed — the problem or constraint>

## Decision
<What was decided>

## Consequences
<What follows from this — tradeoffs, things enabled, things ruled out>
```

**What warrants an ADR:**
- Technology/dependency choices (e.g., "use git2 over shelling out to git")
- Architectural patterns (e.g., "Streamable HTTP only, no stdio")
- Security decisions (e.g., "no tokens in CLI args")
- Decisions where alternatives were seriously considered and rejected

**What does NOT warrant an ADR:**
- Obvious defaults (using serde for serialization in Rust)
- Formatting/style choices covered by linters
- Temporary scaffolding decisions that will be revisited

Write ADRs concisely — 5-15 lines total. The value is in recording *why*, not in
being thorough. If the plan discussion already captured the rationale, distill it.

ADRs themselves are written as their own jj change (a small commit at the bottom
of the stack). This keeps them in version control alongside the work they describe
without polluting the implementation layers.

## Phase 2: Implement

Dispatch implementation sub-agents (model: sonnet) — one per layer in the plan.

Each sub-agent works in isolation — it gets its layer's plan, conventions, and relevant
file paths, then writes the code. See [references/implementation-guide.md](references/implementation-guide.md)
for the detailed prompt template and conventions checklist passed to this agent.

**Critical jj convention**: each layer is its own jj change. The coordinator runs
`jj new` between layers to create the chain:

```bash
jj new -m "Layer 1: <intent>"
# dispatch sub-agent for layer 1
# sub-agent edits files; jj snapshots automatically
jj describe -m "<final commit message for layer 1>"
jj new -m "Layer 2: <intent>"
# dispatch sub-agent for layer 2
# ...
```

The implementation agent does not run `jj` commands — the coordinator manages the
chain. The agent just edits files; jj's automatic snapshotting captures the work
into the current change.

Key constraints for the implementation agent:
- Follow the plan from Phase 1 — diverge only when engineering judgment requires it,
  and document why
- Use Serena's symbolic editing tools (`replace_symbol_body`, `insert_after_symbol`)
  for precise modifications when appropriate
- Write or update tests alongside implementation
- Run `cargo fmt` and `cargo clippy -- -D warnings` (or equivalent) before handing off —
  formatting and lint issues are the implementation agent's responsibility, not the quality agent's
- Do not run tests — that's Phase 3's job
- Do not run jj commands — the coordinator manages the change chain

After all layers complete, briefly summarize what was implemented per layer.

### Per-layer size check

After each layer completes, check the size of that layer's change:
```bash
jj diff --stat -r @ | tail -1
```
If a single layer exceeds ~500 lines, pause and present the user with:
1. The layer's net LOC added/removed
2. A proposed split — which file groups or functional areas could be separate layers
3. The option to proceed as-is if splitting doesn't make sense

Large single-layer diffs compound review rounds. The whole point of stacking is to
keep each layer reviewable.

## Phase 3: Quality

Dispatch a **quality sub-agent** (model: sonnet) to verify the changes. This agent's
context is fresh — it has no bias from having written the code.

See [references/quality-checklist.md](references/quality-checklist.md) for the language-specific
checks. The quality agent:

1. Verifies formatting (`cargo fmt -- --check` / equivalent) — the implementation agent
   should have already fixed these, but verify. If failures remain, fix them in a new
   jj change at the top of the stack (do not amend earlier layers).
2. Verifies lint (`cargo clippy -- -D warnings` / equivalent) — same as above.
3. Runs the test suite (`cargo nextest run --workspace` / equivalent) against `@`,
   which has the full stack applied.
4. **Per-layer build check** (for stacks of 2+ layers): for each layer, check out
   that change with `jj edit <change>`, verify it builds (`cargo check`), then
   `jj edit @` (or the original tip) to return. This catches "layer 2 doesn't
   compile without layer 3" gaps that break the stacked-review story.
5. If any step fails: diagnose, fix, and re-run. Fixes go in a new change at the
   top of the stack — do not amend earlier layers, that loses the review history.
6. Checks the diff for:
   - Unnecessary `.clone()`, `.unwrap()`, `.expect()` (Rust)
   - Dead code introduced or left behind
   - Missing error propagation
   - Test coverage gaps for new behavior

Output: pass/fail with details on any issues found and fixed. If a per-layer build
check fails, that's a structural issue with the stack decomposition — surface it
to the user, who may want to restructure rather than just push fixups.

If the quality agent reports unfixed issues, present them to the user with options.

## Phase 4: Code review

Invoke the `/code-review` skill with `stack` scope (the default). This runs three parallel
sub-agents (correctness, design, architecture+security) and produces deduplicated, verified
findings. The `/code-review` skill is the single source of truth for review methodology — do
not duplicate its logic here.

```
/code-review stack
```

For multi-layer stacks, `/code-review` defaults to per-change review, surfacing findings
scoped to specific layers. This matches how the PRs will be reviewed externally.

The code-review skill will post findings to PRs if they exist (rare at this point —
usually Phase 5 hasn't run yet) or display in-session. Collect the findings from the
review output.

If there are any findings (P1, P2, or P3), present them to the user, then proceed to
Phase 4.5. All severity levels are addressed — P3 is a priority signal, not a skip signal.
If there are zero findings, skip to Phase 5.

## Phase 4.5: Fix and re-review (iterate until clean)

When Phase 4 produces findings:

1. **Decide where each fix belongs.** For each finding, identify which layer in the
   stack it logically belongs to. Findings about layer 1's code should be fixed in
   a change that gets squashed *into* layer 1; findings about layer 2 belong in
   a change for layer 2; etc.

2. **Implement fixes as new changes at the top of the stack.** Dispatch an
   **implementation sub-agent** (model: sonnet) to address all findings.

   The fixes go on top of the current stack as one or more new changes. Don't try
   to edit earlier layers in place yet — that's the *next* step.

   The sub-agent receives:
   - The **original plan from Phase 1** and any ADRs written in Phase 1.5 — this preserves
     architectural intent so fixes don't diverge from the design
   - The full list of P1, P2, and P3 findings with file locations, the layer each
     belongs to, and suggested fixes
   - The same conventions and project context as Phase 2

3. **Squash fixes into their target layers.** After the fix change is verified, the
   coordinator runs:
   ```bash
   jj squash --from <fix-change> --into <target-layer>
   ```
   This moves the fix's content into the layer it belongs to, leaving descendants
   unchanged. jj automatically rebases descendants if there are conflicts.

   For findings that span multiple layers, split the fix change first with
   `jj split` so each piece can be squashed into its correct target.

4. **Verify the build still passes** with the quality sub-agent (Phase 3 logic).

5. **Re-review incrementally.** Run `/code-review stack --since <change-id-of-last-review-tip>`.
   This scopes the review to only what's changed since the last review round, using jj
   change IDs as anchors. Change IDs are stable across rebases, so this works
   correctly even after the squash-into-earlier-layer step.

   Record the change ID at `@` before each review round so you can pass it as
   `--since` to the next round.

6. **Loop**: if the re-review produces new findings, repeat from step 1.
   Present each iteration's findings to the user.

**Circuit breaker**: if 3 iterations haven't converged to a clean review, stop and
present the remaining findings to the user. Something structural needs human judgment.

## Phase 5: Land the stack

After all phases pass (review is clean):

### 5a. Summary
1. Summarize the stack: list each layer's intent and change ID
2. Note files changed per layer
3. Note any deferred decisions or follow-up work

### 5b. Pre-land quality gate
1. Verify formatting, lint, and tests are still clean
2. **Rust projects**: run `cargo doc --no-deps` before landing. This verifies that
   documentation builds cleanly — doc warnings or errors must be fixed before proceeding.
   Doc fixes go in a new change at the top, then squashed into the layer they belong to
   (same pattern as Phase 4.5).
3. Confirm `jj log -r 'stack()'` shows the expected chain

### 5c. Fetch and rebase

Before any pushing, sync the stack with current `main`. This is unconditional —
main may have moved due to a PR merge (yours, an external contributor's, or
another concurrent session's), and your stack needs to land on top of whatever's
there now.

```bash
jj git fetch
```

Determine the bottom of the stack. For each layer's bookmark (in stack order),
check the PR state:

```bash
gh pr view <bookmark-name> --json state,reviewDecision --jq '{state, reviewDecision}'
```

The first bookmark whose PR is **not** in `MERGED` state is the current bottom.
Layers below that point are content-redundant with main (their squash commits
are already in main).

Rebase the stack from the bottom:

```bash
jj rebase -s pr/<bottom-of-stack-slug> -d main
```

`jj rebase -s <change> -d <dest>` rebases the named change *and all its
descendants* — the rest of the stack comes along automatically.

For each merged-out layer below the new bottom, abandon the local orphan:

```bash
jj abandon <old-change-id-of-merged-layer>
```

This is optional cleanup; jj's GC eventually removes the orphan, but abandoning
keeps the visible log uncluttered.

**Always narrate the rebase**, even when clean:

```
Fetched origin (main advanced by N commits since last fetch).
Rebased K changes onto new main.
Abandoned J merged layers from previous stack.
```

If the rebase produced conflicts, surface them — jj records conflicts in the
commit rather than blocking, but the user needs to resolve before continuing.

### 5d. Set bookmarks for each layer

For each change in the rebased stack, set a `pr/<slug>` bookmark. The slug
should reflect the layer's intent, not its position. Use `jj bookmark set`
(which moves an existing bookmark or creates a new one):

```bash
jj bookmark set pr/<slug-1> -r <change-id-1>
jj bookmark set pr/<slug-2> -r <change-id-2>
# ...
```

If a bookmark already exists for a layer (re-running `/develop land` on an
updated stack), this moves it to the new change ID. jj's automatic descendant
rebase already kept the layer's identity intact through any earlier rewrites.

The slugs come from the layer descriptions in the plan. Ask the user to confirm
or override before proceeding the first time; reuse on subsequent runs.

### 5e. Approval-state check

Before pushing, check whether any of the stack's bookmarks have an approved PR.
For each bookmark with an existing PR:

```bash
gh pr view <bookmark-name> --json reviewDecision --jq '.reviewDecision'
```

If any returns `APPROVED`, warn the user:

```
PR #<num> (pr/<slug>) is already approved. Pushing will dismiss the approval
and require re-review. Continue? (y/N)
```

The user can confirm to proceed or abort to handle the change differently
(e.g., land the approved PR first, then push the rest).

### 5f. Push the stack

```bash
jj git push --bookmark pr/<slug-1> --bookmark pr/<slug-2> --bookmark pr/<slug-3>
```

jj's push semantics check that the remote tip matches what jj last knew about.
If a teammate has pushed to one of these bookmarks (uncommon for personal `pr/*`
bookmarks but possible), jj refuses and asks you to fetch first.

### 5g. Open or update PRs

For each bookmark, check if a PR already exists:

```bash
gh pr list --head <bookmark-name> --state open --json number,url,baseRefName
```

For each layer:

**If no PR exists**: create one with the correct base.
- Bottom of stack (layer 1): `--base main --head pr/<slug-1>`
- Each subsequent layer: `--base pr/<slug-N-1> --head pr/<slug-N>`

```bash
gh pr create --base <base> --head <head> --title "<title>" --body "<body>"
```

**If a PR exists**: verify the base is correct (the previous bookmark in the stack,
or `main` for the bottom). If it's wrong (e.g., a stack reorder, or the previous
layer just merged so this layer should now base on `main`), update it:

```bash
gh pr edit <N> --base <correct-base>
```

### 5h. Stack metadata in PR bodies

After all PRs exist, generate the stack footer for each PR body. The footer is
delimited by `---` and a header so it can be detected and replaced on subsequent
runs without disturbing the rest of the body:

```
---
**Stack** (N of M)
- #<num> — pr/<slug>: <title>
- **#<num> — pr/<slug>: <title>** (this PR)
- #<num> — pr/<slug>: <title>

Generated by /land. Re-run /develop land to refresh.
```

For each PR, replace any existing footer (matching `---\n**Stack**` through end of
body) with the regenerated one, then `gh pr edit <N> --body <new-body>`.

### 5i. Branch protection (first PR in repo only)
On the first PR for a new repo, check if main has branch protection rulesets:
```bash
gh api repos/{owner}/{repo}/rulesets --jq 'length'
```
If `0` (no rulesets), create them per [references/repo-setup.md](references/repo-setup.md).
This is a one-time setup — skip on subsequent runs.

### 5j. Tracking
1. Check if a tracking issue exists in `butterflyskies/tasks` for this work
2. If not, create one: `gh issue create --repo butterflyskies/tasks --title "<work description>"`
3. Comment on the tracking issue with each PR link in the stack and a brief status update
4. Assign the issue to the current milestone if one exists

## Re-running /develop land on an updated stack

If the stack was rewritten after initial landing (review feedback addressed,
layers reordered, fixes squashed into earlier layers), running `/develop land` again:

1. Fetches and rebases (Phase 5c) — picks up any merged PRs and advances of main
2. Detects existing bookmarks via `jj log -r 'stack() & remote_bookmarks()'`
3. Moves bookmarks to their new positions (the `pr/<slug>` identity stays even
   though change IDs may have shifted)
4. Checks approval state (Phase 5e) — warns before dismissing approvals
5. Pushes the updated stack (Phase 5f)
6. Updates PR bases if positions changed in the stack (Phase 5g)
7. Regenerates stack footers in all PR bodies (Phase 5h)

## Language detection

Detect project language from:
1. `Cargo.toml` → Rust (load [references/rust.md](references/rust.md))
2. `package.json` → TypeScript/JavaScript
3. `go.mod` → Go
4. `pyproject.toml` / `setup.py` → Python
5. Serena project config (`.serena/project.yml`)

Pass the language-specific reference to sub-agents that need it.

## Model selection

| Sub-agent | Model | Rationale |
|-----------|-------|-----------|
| Planning | **opus** (4.6) | Resolves ambiguity, weighs tradeoffs, asks the right questions, decomposes into stack layers |
| Implementation | **sonnet** | Concrete execution from a well-defined plan |
| Quality | **sonnet** | Mechanical verification — run tools, fix what fails |
| Architectural review (via /code-review) | **opus** (4.6) | Judgment-heavy — completeness gaps, subtle contract breaks, "this will hurt later" |

The coordinator inherits the user's session model (typically opus). Use the `model` parameter
on the Agent tool to set each sub-agent's model explicitly.

## Guidelines

- **Never amend.** All work goes into new jj changes. The coordinator manages the chain
  via `jj new` between layers, `jj squash --from --into` to fold fixes into earlier
  layers when needed.
- **Always fetch and rebase before pushing.** Phase 5c is unconditional. Main may have
  moved; your stack needs to be on top of current main, not stale main.
- **Sub-agents are disposable context** — don't hesitate to spawn them. The cost is
  tokens, not your coordinator context window.
- **Fail fast** — if Phase 1 reveals the task is unclear, stop and ask. Don't send
  ambiguity downstream.
- **Trust sub-agent output but verify P1s** — for critical findings, spot-check by
  reading the relevant code yourself before presenting to the user.
- **No gold-plating** — implement what was asked, nothing more. If you see an improvement
  opportunity, mention it; don't do it.
- **Stacks are intent, not process.** A two-layer stack where layer 2 is a one-line
  fixup is a sign the layer should have been part of layer 1 (squash it in) or doesn't
  warrant a separate PR (squash into the parent before landing).
