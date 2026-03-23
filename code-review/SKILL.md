---
name: code-review
description: "Systematic code review with sub-agent analysis. Works on PRs, branches, files, or staged changes."
---

# /code-review — Systematic Code Review

Perform a structured, multi-phase code review. This skill produces actionable findings
with severity, location, and concrete fixes — not style nits or praise.

Use memory-mcp's `read` tool to load the `code-review-patterns` memory (scope: global)
before starting. It contains learned patterns from previous reviews that should inform
what you look for. If the memory doesn't exist yet, proceed without it — findings from
this review will seed it.

## Argument handling

`$ARGUMENTS` determines the review scope:

| Argument             | Scope                                         |
|----------------------|-----------------------------------------------|
| *(empty)*            | All uncommitted changes (staged + unstaged)   |
| `pr` or `pr <N>`     | Current branch's PR diff, or PR #N            |
| `branch`             | All commits on current branch vs base          |
| `file <path>`        | Single file, full review                      |
| `files <glob>`       | Multiple files matching pattern               |
| `commit <ref>`       | Single commit diff                            |
| `--since <ref>`      | Incremental: only commits since `<ref>`       |

### Incremental review mode (`--since`)

When `--since <commit-sha>` is appended to any scope argument (e.g., `branch --since abc123`),
the review operates in **incremental mode** for fix-round efficiency:

1. The **primary diff** is limited to commits after `<ref>` — this is what sub-agents analyze
2. Sub-agents also receive a **prior findings summary** — the findings from the previous round,
   with their status (fixed, still-open, or deferred-with-rationale)
3. Sub-agents are instructed to:
   - Verify each prior finding is addressed (fixed in the new commits or explicitly deferred)
   - Flag **new issues** introduced by the fix commits
   - Only cross-reference unchanged code when the new changes directly affect it
   - Not re-review code that hasn't changed since the last review
4. The coordinator merges the incremental findings with the prior findings to produce a
   cumulative status report

This mode cuts review time on rounds 2+ significantly by preventing re-analysis of
already-reviewed, unchanged code. The `/develop` skill's Phase 4.5 should use this mode
automatically when re-invoking `/code-review` after fixes.

## Phase 1: Gather context

Before reviewing code, build understanding. This phase is **silent** — no output to user.

1. **Identify the diff** — resolve `$ARGUMENTS` to a concrete set of changed files and hunks
2. **Read project conventions** — check for `.claude/CLAUDE.md`, memory-mcp project memories
   (use `list` filtered by project scope, look for `project-overview`, conventions), and
   any linter/formatter configs
3. **Understand architecture** — for non-trivial changes, use Serena's `get_symbols_overview`
   on affected files to understand the surrounding code structure. Read symbol bodies only
   when needed to understand how changed code fits into the system.
4. **Trace callers** — for any function/method whose signature, behavior, or error handling
   changed, use `find_referencing_symbols` to identify all call sites. This is critical for
   catching breakage that looks fine in isolation.

**Context budget:** aim for roughly 1:1 ratio of changed code to surrounding context.
More context than code means you're over-reading. Less means you're likely missing impact.

**Large diff warning:** research shows review effectiveness drops sharply past 400 lines
of changed code. If the diff exceeds ~500 lines, tell the user upfront and suggest
reviewing in logical chunks (by file group or functional area) rather than all at once.

## Phase 2: Analyze

Launch all three sub-agents in a **single message** with three parallel Agent tool calls,
each with `run_in_background: true`. This ensures true concurrent execution — launching
them sequentially wastes time and defeats the purpose of independent analysis. Each agent
gets the same diff and context but a different analytical lens. The separation ensures
independent findings — a bug one agent normalizes, another catches. Use **sonnet** for
sub-agents A and B (mechanical analysis), **opus** for sub-agent C (judgment-heavy
architectural review).

### Sub-agent A: Correctness & Safety

```
You are reviewing code changes for correctness and safety issues. Precision matters
more than count. Every genuine finding at any priority level (P1, P2, or P3) is
valuable and will be addressed. False positives waste verification time and erode
trust in the review process — a finding that isn't real is worse than a finding
you didn't report.

Review the following changes for:

**Logic errors**
- Off-by-one, wrong operator, inverted condition, missing early return
- Race conditions, TOCTOU, shared mutable state
- Error handling: swallowed errors, wrong error type, missing propagation

**Completeness gaps** (this is the #1 thing LLM reviewers miss — be thorough)
- For each branch/match arm: what cases exist in the domain that aren't covered?
- For each identifier used in dedup/lookup/fallback: what entity types share that
  format? (e.g., owner/repo#77 matches issues, PRs, AND discussions)
- For each pattern match on input: trace all callers — what inputs reach this code
  that DON'T match any handled pattern?
- "Inputs always look like X" is a red flag — verify by tracing actual data flow
- For each resource created (sessions, connections, handles, caches, temp files):
  what cleans it up? Timeout, eviction, explicit close, Drop impl? If nothing
  cleans it up, that's a finding.

**Data integrity**
- Mutations that could corrupt or lose data (wrong UPDATE scope, missing WHERE clause)
- LIKE/GLOB wildcards in user-supplied values without escaping
- Identifier collisions across entity types sharing the same format
- Upsert/dedup logic that collapses things that should remain distinct

For each finding, output EXACTLY this format:
**[P1|P2|P3] <short title>**
- File: `<path>:<line>`
- Issue: <1-2 sentence description of what's wrong>
- Impact: <what breaks, and under what conditions>
- Fix: <concrete code change or approach>
```

### Sub-agent B: Design & Maintainability

```
You are reviewing code changes for design and maintainability issues. Precision matters
more than count. Every genuine finding at any priority level (P1, P2, or P3) is
valuable and will be addressed. False positives waste verification time and erode
trust in the review process — a finding that isn't real is worse than a finding
you didn't report.

Review the following changes for:

**API contract issues**
- Breaking changes to public interfaces without migration
- Inconsistent naming, parameter ordering, or return types vs existing patterns
- Missing or misleading error messages that will confuse callers

**Dead code & redundancy**
- Code that the change made unreachable or unnecessary
- Duplicated logic that should be consolidated
- Imports, variables, enum variants, or parameters that are no longer used
- For `Option`-guarded features: does disabling the feature leave allocated-but-unused
  fields? If so, group the feature's state into a sub-struct and wrap in `Option`.

**Testing gaps**
- Changed behavior that has no corresponding test update
- Edge cases in new code that tests don't cover
- Test assertions that don't actually verify the behavior they claim to test

**Test quality**
- Do tests exercise the library's public API, or do they duplicate internal logic?
  Tests that reimplement the production code path instead of calling it prove nothing
  about the real code.
- Are assertions non-vacuous? A test should fail if its assertion is removed. Tests
  that compare single-element collections, assert `true`, or check trivially-true
  conditions waste CI time and give false confidence.
- For edge-case tests: is the edge case actually exercised? Trace the test input
  through the code — does it actually hit the branch/condition the test name claims?

**Architectural fit**
- Does this change follow the project's established patterns?
- Are abstractions at the right level? (over-engineering is as bad as under-)
- Will this change make future work harder? (coupling, hidden dependencies)

For each finding, output EXACTLY this format:
**[P1|P2|P3] <short title>**
- File: `<path>:<line>`
- Issue: <1-2 sentence description of what's wrong>
- Impact: <what breaks, and under what conditions>
- Fix: <concrete code change or approach>
```

### Sub-agent C: Architecture & Security (model: opus)

```
You are reviewing code changes for architectural fitness and security. You did NOT
write this code and have NOT seen the implementation process — only the diff and
the project structure. This separation is intentional: you catch things the
implementer normalized away. Precision matters more than count. Every genuine
finding at any priority level (P1, P2, or P3) is valuable and will be addressed.
False positives waste verification time and erode trust in the review process — a
finding that isn't real is worse than a finding you didn't report.

Review the following changes for:

**API contracts**
- Are changes backward-compatible? If breaking: are ALL callers updated?
  Use `find_referencing_symbols` to verify.
- Are error types consistent with the project's conventions?
- Would a consumer of this API be surprised by the new behavior?

**Architectural fit**
- Does this follow established patterns in the codebase?
- If introducing a new pattern: is the old pattern being migrated, or will both coexist?
- Are dependencies flowing in the right direction? (no circular deps, no upward deps)
- Is the abstraction level appropriate? (not over-engineered, not under-abstracted)

**Completeness**
- For each match/branch: what cases exist in the domain that aren't handled?
- For each input path: trace what data can actually arrive — are all shapes covered?
- Are error paths tested? Is the happy path the only path tested?

**Security (STRIDE threat model)**
For each change that touches trust boundaries, data flows, or auth:
- Spoofing: can an attacker impersonate a user or system?
- Tampering: can data be modified in transit/at rest without detection?
- Repudiation: can actions occur without accountability/logging?
- Information disclosure: can sensitive data leak via logs, errors, side channels?
- Denial of service: can an attacker exhaust resources (CPU, memory, connections)?
  Specifically: stateful services that accept external connections — is there a
  session/connection limit and idle timeout? Unbounded accumulation from external
  triggers is a resource exhaustion vector.
- Elevation of privilege: can a user gain access beyond their authorization?

Also check for concrete injection vectors:
- SQL, command, XSS, template, LDAP, path traversal
- Hardcoded secrets, weak crypto, insufficient randomness

**Security (beyond STRIDE)**
- Credential exposure: can secrets appear in process listings (ps), logs, stdout,
  error messages, stack traces, or debug output? Check CLI args, Display/Debug impls,
  and tracing instrumentation on structs that hold secrets.
- Secrets in git: are tokens, keys, or credentials hardcoded or at risk of being
  committed? Check for missing .gitignore entries, secrets in config files, or test
  fixtures containing real credentials.
- Trust boundaries: where does external input enter the system? Is it validated before
  use? Check HTTP handlers, MCP tool parameters, file paths from user input (path
  traversal), and deserialized data from untrusted sources.
- Auth bypass: can any code path skip authentication or authorization? Trace from the
  network entry point to the protected operation — is there a path that doesn't check
  credentials?
- Information leakage: do error responses reveal internal structure (stack traces, file
  paths, SQL queries, dependency versions) to external callers?
- Dependency surface: do new dependencies introduce known vulnerabilities or excessive
  privilege? Flag any dependency that pulls in native code, network access, or filesystem
  access beyond what the feature requires.

**Simplicity**
- Could the same result be achieved with less code?
- Are there intermediate abstractions that exist only to serve this one use case?
- Is there dead code from a previous approach that should be cleaned up?
- Would a future reader understand this without the PR description?

For each finding, output EXACTLY this format:
**[P1|P2|P3] <short title>**
- File: `<path>:<line>`
- Issue: <1-2 sentence description of what's wrong>
- Impact: <what breaks, and under what conditions>
- Fix: <concrete code change or approach>
```

### Providing context to sub-agents

Each sub-agent receives:
1. The diff (changed lines with surrounding context)
2. Project conventions (from Phase 1)
3. Symbol overview of affected files
4. Caller information for changed function signatures
5. Contents of `code-review-patterns` memory from memory-mcp (learned patterns)
6. **Previously dismissed findings** (for multi-round reviews only — see Phase 3)

Use Serena tools within sub-agents for any additional code exploration needed.

When running a subsequent round on the same scope, include a "Previously dismissed"
section in each sub-agent prompt. This prevents agents from re-discovering the same
false positives and wasting verification cycles. Use this format:

```
Previously evaluated and dismissed (do not re-flag unless you have NEW evidence
that changes the analysis):
- "<finding title>" — <reason for dismissal>
```

Only include findings that were **verified as false positives** in a prior Phase 3 —
not findings that were real and fixed (those belong in the "previously fixed" context).

### Severity definitions

| Level | Meaning                                                    |
|-------|------------------------------------------------------------|
| P1    | Data loss, security vulnerability, crash, or silent corruption |
| P2    | Incorrect behavior, broken edge case, or test gap            |
| P3    | Design issue, dead code, or maintainability concern          |

## Phase 3: Deduplicate & verify

After all three sub-agents return:

1. **Merge findings** — combine all three agents' results, removing duplicates
2. **Verify each finding** — for every P1 and P2, read the actual code to confirm
   the issue is real. LLM reviewers hallucinate findings; do not pass through
   unverified claims. Drop any finding you cannot confirm by reading the code.
3. **Check for false positives** — common traps:
   - "Missing error handling" when the framework/caller already handles it
   - "SQL injection" when parameterized queries are actually used
   - "Race condition" in single-threaded or async-but-sequential code
   - "Missing null check" when the type system prevents nulls
   - "Unused variable" that is used in a macro, template, or framework convention
   - "Breaking change" to an internal API with no external consumers
   - Flagging an intentional pattern as a bug (check if the same pattern appears
     elsewhere in the codebase — if so, it's likely deliberate). However: if a
     pattern *looks* like a bug but appears intentional, still flag it as a P3
     asking the author to add a comment explaining why. Code that requires
     reviewer investigation to distinguish from a bug needs a comment.
4. **Track dismissed findings** — for each finding dropped as a false positive,
   record the finding title and a one-line reason for dismissal. This list is
   carried forward into sub-agent prompts in subsequent rounds (see Phase 2,
   "Providing context to sub-agents") so agents do not re-flag the same non-issues.
   Include dismissed findings in the report's Summary section for transparency.

## Phase 4: Report

Present findings grouped by severity, then by file. Use this format:

```
## Code Review: <scope description>

### P1 — Critical (N findings)

**<title>** — `path/to/file.py:42`
<description with concrete fix>

### P2 — Important (N findings)

...

### P3 — Suggestions (N findings)

...

### Summary
- N files reviewed, M findings (X P1, Y P2, Z P3)
- Key themes: <1-2 sentence synthesis of what the findings reveal>
```

If there are zero findings at a severity level, omit that section entirely.
If there are zero findings total, say so clearly — don't invent issues to fill space.

## Phase 5: Post findings

Findings should be captured somewhere durable, not just displayed in-session. Try each
option in order and use the first that works:

### Option A: Post to an existing PR

Check if the reviewed scope corresponds to a PR:
- If `$ARGUMENTS` is `pr` or `pr <N>`, the PR is already known
- If `$ARGUMENTS` is `branch`, check for an open PR on the current branch:
  `gh pr list --head <branch> --state open --json number,url`
- If a PR exists, post the review as a PR comment: `gh pr comment <N> --body <review>`

### Option B: Post to an existing issue

If no PR exists, check if there's a tracked issue for the work:
- Check Serena project memories for issue references
- Check gh-notify work items for linked issues
- If an issue exists, post findings as a comment: `gh issue comment <N> --repo <repo> --body <review>`

### Option C: Create a PR

If the work is in a git repo with uncommitted or unpushed changes on a feature branch:
1. Ensure changes are committed and pushed
2. Create a PR: `gh pr create --title "<branch context>" --body <review>`
3. The review becomes the PR description

Only do this when it's straightforward — the branch exists, changes are committed, and
it's clear what the PR should be. Ask the user if anything is ambiguous.

### Option D: Create an issue

If the work is in a repo but there's no PR or existing issue:
- Create an issue with the review findings: `gh issue create --title "Code review: <scope>" --body <review>`
- This captures findings for later action

### Option E: Display in-session only

If none of the above apply (e.g., reviewing files outside version control, or the user
is working interactively and will act on findings immediately), displaying the review
in the conversation is sufficient. The findings are still valuable even without a
durable destination.

Always tell the user where the review was posted (PR URL, issue URL, or "displayed in-session").

## Phase 6: Learn (optional)

If the review produced P1 or P2 findings that reveal a **pattern** (not just a one-off bug),
update the `code-review-patterns` memory in memory-mcp (scope: global):

- New pattern: what to look for, why it matters, example from this review
- Refinement: if an existing pattern helped catch something, note the confirmation
- Removal: if a pattern consistently produces false positives, remove it

This is the feedback loop — each review makes future reviews better.

Only update patterns for findings that were **verified** in Phase 3. Do not add
speculative patterns from unconfirmed findings.

## Configuration

The skill respects project-level overrides. If a project's memory-mcp memories contain a
`code-review-config` memory (check with `list` filtered by project scope), read it and apply:

- **skip_categories**: list of categories to suppress (e.g., `["dead_code"]`)
- **extra_patterns**: additional domain-specific patterns to check
- **severity_overrides**: reclassify certain finding types (e.g., `dead_code: P3→ignore`)

## Guidelines

- **No style nits** — formatting, naming conventions, and whitespace are for linters, not review
- **No praise** — "great use of X" wastes everyone's time
- **Concrete fixes only** — every finding must include a specific fix, not "consider refactoring"
- **Assume competence** — if the author demonstrates a practice correctly elsewhere in the
  diff (e.g., parameterized queries in 9 out of 10 call sites), the 10th omission is a real
  finding. But explaining *what* parameterized queries are is not — the author clearly knows.
  Focus on what they missed, not what they already understand. Generic tutorial-style advice
  erodes trust in the review and makes developers skim past the findings that matter.
- **Verify before reporting** — a false positive is worse than a missed finding, because it
  erodes trust in the review process
