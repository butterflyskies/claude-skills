---
name: develop
description: "End-to-end development workflow with sub-agent specialization. Use when implementing features, fixing bugs, or making code changes that should follow project standards. Dispatches focused sub-agents for planning, implementation, quality checks, and architectural review — keeping the coordinator lean and the overarching goals visible."
---

# /develop — Sub-Agent Development Workflow

Implement changes using specialized sub-agents, each with a dedicated context window.
The coordinator (you) stays lean — orchestrate, don't accumulate.

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
| `quality` | Phase 3 only — check quality of current changes |
| `review` | Phase 4 only — architectural review of current changes |
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

## Phase 1: Plan

Dispatch a **planning sub-agent** (model: opus) to:

1. If `$ARGUMENTS` references an issue, fetch it: `gh issue view <N> --json title,body,comments`
2. Read the relevant code using Serena's symbolic tools — `get_symbols_overview` for structure,
   `find_symbol` with `include_body=true` only for symbols that need modification
3. Identify all files and symbols that need to change
4. For each changed function/method signature, use `find_referencing_symbols` to find callers
5. For stateful subsystems: identify resource lifecycle (creation → cleanup → limits).
   External connections/sessions require a timeout and max-count strategy in the plan.
6. Propose an approach: what changes, in what order, and why
7. Flag risks, ambiguities, or decisions that need user input

**Sub-agent prompt template:**
```
You are a planning agent. Your job is to understand the task and propose a concrete
implementation approach. Do NOT write code — produce a plan.

Task: <task description>
Project language: <language>
Build command: <from CLAUDE.md or Cargo.toml etc.>
Project conventions: <from memory-mcp project memories if available>

Use Serena's symbolic tools to explore the codebase efficiently:
- get_symbols_overview for file structure
- find_symbol with include_body=true only for symbols you need to understand deeply
- find_referencing_symbols for impact analysis

Output format:
## Plan
1. [Change description] — `file:symbol`
   - Why: [rationale]
   - Impact: [callers/dependents affected]
2. ...

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

## Phase 2: Implement

Dispatch an **implementation sub-agent** (model: sonnet) with the plan from Phase 1.

The sub-agent works in isolation — it gets the plan, conventions, and relevant file paths,
then writes the code. See [references/implementation-guide.md](references/implementation-guide.md)
for the detailed prompt template and conventions checklist passed to this agent.

Key constraints for the implementation agent:
- Follow the plan from Phase 1 — diverge only when engineering judgment requires it,
  and document why
- Use Serena's symbolic editing tools (`replace_symbol_body`, `insert_after_symbol`)
  for precise modifications when appropriate
- Write or update tests alongside implementation
- Run `cargo fmt` and `cargo clippy -- -D warnings` (or equivalent) before handing off —
  formatting and lint issues are the implementation agent's responsibility, not the quality agent's
- Do not run tests — that's Phase 3's job

After the sub-agent returns, briefly summarize what was implemented.

### Diff-size check

After the implementation agent returns, check the size of the changes:
```
git diff --stat | tail -1
```
If the net change exceeds ~500 lines, pause and present the user with:
1. The total LOC added/removed
2. A proposed split (by file group or functional area)
3. The option to proceed as-is if splitting doesn't make sense

Large diffs compound review rounds — a 1000-line PR averages 4+ review rounds while
a 200-line PR typically converges in 1-2.

## Phase 3: Quality

Dispatch a **quality sub-agent** (model: sonnet) to verify the changes. This agent's
context is fresh — it has no bias from having written the code.

See [references/quality-checklist.md](references/quality-checklist.md) for the language-specific
checks. The quality agent:

1. Verifies formatting (`cargo fmt -- --check` / equivalent) — the implementation agent
   should have already fixed these, but verify. If failures remain, fix them.
2. Verifies lint (`cargo clippy -- -D warnings` / equivalent) — same as above.
3. Runs the test suite (`cargo nextest run --workspace` / equivalent)
4. If any step fails: diagnose, fix, and re-run. Report what was fixed.
5. Checks the diff for:
   - Unnecessary `.clone()`, `.unwrap()`, `.expect()` (Rust)
   - Dead code introduced or left behind
   - Missing error propagation
   - Test coverage gaps for new behavior

Output: pass/fail with details on any issues found and fixed.

If the quality agent reports unfixed issues, present them to the user with options.

## Phase 4: Code review

Invoke the `/code-review` skill with `branch` scope. This runs three parallel sub-agents
(correctness, design, architecture+security) and produces deduplicated, verified findings.
The `/code-review` skill is the single source of truth for review methodology — do not
duplicate its logic here.

```
/code-review branch
```

The code-review skill will post findings to the PR if one exists, or display in-session.
Collect the findings from the review output.

If there are any findings (P1, P2, or P3), present them to the user, then proceed to
Phase 4.5. All severity levels are addressed — P3 is a priority signal, not a skip signal.
If there are zero findings, skip to Phase 5.

## Phase 4.5: Fix and re-review (iterate until clean)

When Phase 4 produces findings:

1. Dispatch an **implementation sub-agent** (model: sonnet) with the findings as its task.
   The sub-agent receives:
   - The **original plan from Phase 1** and any ADRs written in Phase 1.5 — this preserves
     architectural intent so fixes don't diverge from the design
   - The full list of P1 and P2 findings with file locations and suggested fixes
   - P3 findings for implementation (these are real findings that should be fixed)
   - The same conventions and project context as Phase 2

2. After fixes are applied, dispatch the **quality sub-agent** (model: sonnet) again
   to verify fmt/clippy/tests still pass.

3. Run `/code-review branch --since <last-reviewed-commit>` to use incremental review
   mode. This scopes the review to only the fix commits, verifies prior findings are
   resolved, and checks for new issues — without re-reviewing unchanged code.
   Record the HEAD commit SHA before each review round so you can pass it as `--since`
   to the next round.

4. **Loop**: if the re-review produces new P1 or P2 findings, repeat from step 1.
   Present each iteration's findings to the user.

**Circuit breaker**: if 3 iterations haven't converged to a clean review, stop and
present the remaining findings to the user. Something structural needs human judgment.

## Phase 5: Land

After all phases pass (review is clean):

### 5a. Summary
1. Summarize what was done (1-3 bullet points)
2. List files changed
3. Note any deferred decisions or follow-up work

### 5b. Commit and push
1. Stage all relevant files (not secrets). For `.serena/`: commit everything that
   `.serena/.gitignore` doesn't exclude — this includes `project.yml` (project config),
   `.gitignore`, and `memories/` (project-scoped knowledge shared across sessions).
   Serena's own `.gitignore` already excludes `cache/` and `project.local.yml`.
2. **Rust projects**: run `cargo doc --no-deps` before committing. This verifies that
   documentation builds cleanly — doc warnings or errors must be fixed before proceeding.
   The generated output in `target/doc/` is not committed (it's gitignored).
3. Commit with a descriptive message following repo conventions
4. Push to the feature branch
5. Use `GIT_CONFIG_GLOBAL=~/.gitconfig.ai` and `GH_CONFIG_DIR=~/.config/gh-butterflysky-ai`
   for all git/gh operations

### 5c. Create PR
1. Create a pull request via `gh pr create`
2. Title: concise description of the change
3. Body: summary from 5a, list of ADRs written, link to any tracking issue
4. If a tracking issue exists, link it in the PR body

### 5d. Branch protection
On the first PR for a new repo, check if main has branch protection rulesets:
```bash
gh api repos/{owner}/{repo}/rulesets --jq 'length'
```
If `0` (no rulesets), create them per [references/repo-setup.md](references/repo-setup.md).
This is a one-time setup — skip on subsequent PRs.

### 5e. Tracking
1. Check if a tracking issue exists in `butterflyskies/tasks` for this work
2. If not, create one: `gh issue create --repo butterflyskies/tasks --title "<work description>"`
3. Comment on the tracking issue with the PR link and a brief status update
4. Assign the issue to the current milestone if one exists

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
| Planning | **opus** | Resolves ambiguity, weighs tradeoffs, asks the right questions |
| Implementation | **sonnet** | Concrete execution from a well-defined plan |
| Quality | **sonnet** | Mechanical verification — run tools, fix what fails |
| Architectural review | **opus** | Judgment-heavy — completeness gaps, subtle contract breaks, "this will hurt later" |

The coordinator inherits the user's session model (typically opus). Use the `model` parameter
on the Agent tool to set each sub-agent's model explicitly.

## Guidelines

- **Sub-agents are disposable context** — don't hesitate to spawn them. The cost is
  tokens, not your coordinator context window.
- **Fail fast** — if Phase 1 reveals the task is unclear, stop and ask. Don't send
  ambiguity downstream.
- **Trust sub-agent output but verify P1s** — for critical findings, spot-check by
  reading the relevant code yourself before presenting to the user.
- **No gold-plating** — implement what was asked, nothing more. If you see an improvement
  opportunity, mention it; don't do it.
