---
name: code-review
description: "Systematic code review with sub-agent analysis. Works on jj revsets, single changes, PRs, or working-copy edits."
---

# /code-review — Systematic Code Review

Perform a structured, multi-phase code review. This skill produces actionable findings
with severity, location, and concrete fixes — not style nits or praise.

Use memory-mcp's `read` tool to load the `code-review-patterns` memory (scope: global)
before starting. It contains learned patterns from previous reviews that should inform
what you look for. If the memory doesn't exist yet, proceed without it — findings from
this review will seed it.

## Argument handling

`$ARGUMENTS` determines the review scope. The default is `stack`, which uses the
revset alias defined in [references/stacking-conventions.md](../references/stacking-conventions.md).

| Argument                    | Scope                                                |
|-----------------------------|------------------------------------------------------|
| *(empty)* or `stack`        | `bookmark_base()..@` — the current stack from last published bookmark |
| `change <id>`               | Single jj change (use the alphabetic change ID)      |
| `range <revset>`            | Any jj revset, e.g. `pr/layer-1..pr/layer-2`         |
| `working`                   | Working-copy diff only — `@-..@`                     |
| `pr` or `pr <N>`            | The change(s) backing the current branch's PR, or PR #N |
| `--since <change-id>`       | Modifier on any scope: limit to descendants of `<change-id>` |

If `bookmark_base()` resolves to nothing (no remote bookmarks in ancestry), fall
back to `trunk()..@` and tell the user this happened.

If the resolved revset is empty, report it and stop. The most common reason is
that `@` is sitting on a published bookmark with no work on top — the user
needs to `jj new` first.

### Incremental review mode (`--since`)

When `--since <change-id>` is appended (e.g., `stack --since abcdefg`), the
review operates in **incremental mode** for fix-round efficiency:

1. The **primary diff** is limited to the descendants of `<change-id>` intersected
   with the base scope. Sub-agents analyze only this subset.
2. Sub-agents also receive a **prior findings summary** — the findings from the
   previous round, with their status (fixed, still-open, or deferred-with-rationale).
3. Sub-agents are instructed to:
   - Verify each prior finding is addressed (fixed in the new commits or explicitly deferred)
   - Flag **new issues** introduced by the fix commits
   - Only cross-reference unchanged code when the new changes directly affect it
   - Not re-review code that hasn't changed since the last review
4. The coordinator merges the incremental findings with the prior findings to produce a
   cumulative status report.

Change IDs are stable across rebases in jj, so `--since` survives `jj squash`,
`jj rebase`, and other history rewrites without breaking. This is materially
better than git-SHA-based incremental review.

The `/develop` skill's Phase 4.5 uses this mode automatically when re-invoking
`/code-review` after fixes.

## Phase 1: Gather context

Before reviewing code, build understanding. This phase is **silent** — no output to user.

1. **Resolve the scope.** Run `jj log -r <revset>` to confirm the revset matches what
   the user intended. For `pr` scope, fetch the PR's commits and resolve to their
   change IDs.
2. **Identify changed files and hunks.** Use `jj diff -r <revset>` for the cumulative
   diff across the scope. For per-change context, `jj diff -r <each-change>`.
3. **Read project conventions** — check for `.claude/CLAUDE.md`, memory-mcp project memories
   (use `list` filtered by project scope, look for `project-overview`, conventions), and
   any linter/formatter configs.
4. **Understand architecture** — for non-trivial changes, use Serena's `get_symbols_overview`
   on affected files to understand the surrounding code structure. Read symbol bodies only
   when needed to understand how changed code fits into the system.
5. **Trace callers** — for any function/method whose signature, behavior, or error handling
   changed, use `find_referencing_symbols` to identify all call sites. This is critical for
   catching breakage that looks fine in isolation.
6. **Load design artifacts** — check for `docs/design/` and `docs/adr/` directories. If
   the change has associated design docs (requirements, test plans, architecture, ADRs),
   read them. These are the contract for *what should exist* — sub-agents need them to
   identify missing coverage, not just bugs in code that's present. Also check for linked
   issues (`gh issue view <N>`) if the branch name or commit messages reference one.
   Pass relevant artifacts to sub-agents as "Design context."

   **Resolving design-vs-code divergence:** when a sub-agent finds that the code doesn't
   match the design docs (or vice versa), don't mechanically pick a winner. Determine
   what the user intended *this change* to accomplish, then judge the divergence in that
   context:

   1. **Establish the change's intent.** Read the PR description, linked issue, commit
      messages, and any conversation context. What was the user trying to build? Is this
      a "implement the spec" change, or a "prototype and iterate" change?

   2. **Classify the divergence:**
      - **Unimplemented requirement** — design says X should exist, code doesn't have it.
        But is it in scope? Check whether the issue or PR description scopes this to a
        subset of the design. Phased implementation is normal — missing requirements are
        only findings if they're in scope for *this* change.
      - **Contradicted requirement** — code does Y where design says X. This could be:
        (a) a bug in the code, (b) a discovery during implementation that invalidates
        the design, or (c) an intentional deviation the user hasn't documented yet.
      - **Stale design** — the design describes an older architecture and hasn't been
        updated to match intentional evolution. Common when design docs are written
        up-front and code legitimately outgrows them.

   3. **Choose severity based on confidence:**
      - If the divergence contradicts explicit in-scope requirements and there's no
        signal the user intended to deviate → P2 finding.
      - If the divergence *might* be intentional (code looks deliberate, or the design
        is old and the code is clearly more evolved) → P3, framed as "design doc may
        be stale — verify intent."
      - If you genuinely can't tell → surface both sides to the user without assuming
        either is correct. Quote the specific requirement and the specific code, and
        ask which reflects current intent.

   Timestamps are a useful *signal* (check with `jj log -r 'ancestors(@)' --no-graph
   -T 'change_id ++ " " ++ committer.timestamp().local_format("%Y-%m-%d")' <path>`)
   but they're not authority. A design doc committed yesterday can still be aspirational
   for a future phase. Code committed today can still be wrong.

**Context budget:** aim for roughly 1:1 ratio of changed code to surrounding context.
More context than code means you're over-reading. Less means you're likely missing impact.

**Large diff warning:** review effectiveness drops sharply past 400 lines of changed
code. If the cumulative diff exceeds ~500 lines, tell the user upfront and suggest
either reviewing per-change (each commit in the stack reviewed independently) or
splitting the stack. For stacks, per-change review is usually the right answer —
that's what the stack structure is *for*.

## Phase 2: Analyze

Launch all three sub-agents in a **single message** with three parallel Agent tool calls,
each with `run_in_background: true`. This ensures true concurrent execution — launching
them sequentially wastes time and defeats the purpose of independent analysis. Each agent
gets the same diff and context but a different analytical lens. The separation ensures
independent findings — a bug one agent normalizes, another catches. Use **sonnet** for
sub-agents A and B (mechanical analysis), **opus** (4.6) for sub-agent C (judgment-heavy
architectural review).

When reviewing a multi-change scope (a stack of N commits), there are two valid approaches:

- **Cumulative**: pass all sub-agents the cumulative diff. Faster, fewer tokens, but
  loses per-change boundaries — a bug introduced in commit 1 and fixed in commit 3
  doesn't surface as either a bug or a fix.
- **Per-change**: dispatch one set of sub-agents per change in the stack. Slower, more
  tokens, but each commit is reviewed as the reviewer will see it on the PR.

Default to **per-change** for stacks of 2 or more, **cumulative** for single-change
scopes. The user can override with `--cumulative` or `--per-change`.

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
- Change: `<jj change ID where the issue lives>`
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

**Stack hygiene** (when reviewing per-change in a stack)
- Does this change belong in this commit? If a fixup belongs in an earlier commit
  in the stack, it should be squashed there before landing.
- Are commits cohesive? Each commit should be one logical change. Mixing refactor
  + new feature in one commit makes review harder and revert riskier.

**Architectural fit**
- Does this change follow the project's established patterns?
- Are abstractions at the right level? (over-engineering is as bad as under-)
- Will this change make future work harder? (coupling, hidden dependencies)

**Design validation** (when design artifacts are provided)
- Cross-reference requirements against implementation: does each requirement (R-01,
  R-02, ...) have corresponding code? Flag requirements with no implementation.
- Cross-reference test plan against tests: does each test case (TC-01, TC-02a, ...)
  have a corresponding test? Flag test cases with no test, and tests that don't
  actually verify what the test case specifies.
- Cross-reference ADR decisions against implementation: does the code match the
  decision? Flag divergences (may be intentional — report, don't assume).
- Check behavioral parity requirements: if the design says "test double must behave
  like production for X," verify the double actually enforces X (e.g., dimension
  validation, error semantics).

For each finding, output EXACTLY this format:
**[P1|P2|P3] <short title>**
- File: `<path>:<line>`
- Change: `<jj change ID where the issue lives>`
- Issue: <1-2 sentence description of what's wrong>
- Impact: <what breaks, and under what conditions>
- Fix: <concrete code change or approach>
```

### Sub-agent C: Architecture & Security (model: opus 4.6)

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

**Design compliance** (when design artifacts are provided)
- Verify ADR decisions are faithfully implemented — if an ADR says "use X pattern,"
  confirm the code uses X, not a variation. Flag divergences as findings.
- Verify requirements coverage — does each requirement have corresponding code?
  Missing requirements are P2 findings.
- Verify architectural diagrams match the actual module/trait/type structure.
  Stale diagrams that don't match code are P3 findings.

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
- Secrets in jj/git: are tokens, keys, or credentials hardcoded or at risk of being
  committed? Check for missing .gitignore entries, secrets in config files, or test
  fixtures containing real credentials. Note: jj snapshots automatically, so a secret
  added to the working copy is *already* in a jj change — `jj abandon` and rewrite
  ancestors if you find one.
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
- Change: `<jj change ID where the issue lives>`
- Issue: <1-2 sentence description of what's wrong>
- Impact: <what breaks, and under what conditions>
- Fix: <concrete code change or approach>
```

### Providing context to sub-agents

Each sub-agent receives:
1. The diff (`jj diff -r <revset>` for cumulative, `jj diff -r <change>` for per-change)
2. The change IDs in scope and their descriptions (`jj log -r <revset>`)
3. Project conventions (from Phase 1)
4. Symbol overview of affected files
5. Caller information for changed function signatures
6. Contents of `code-review-patterns` memory from memory-mcp (learned patterns)
7. **Design artifacts** (from Phase 1, step 6) — requirements, test plan, ADRs, and
   architecture docs when they exist. Sub-agents B and C use these to validate
   implementation completeness: does each requirement have corresponding code? Does
   each test case in the plan have a corresponding test? Do ADR decisions match the
   implementation? This catches **missing coverage** that code-only review cannot.
8. **Previously dismissed findings** (for multi-round reviews only — see Phase 3)

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

1. **Merge findings** — combine all three agents' results, removing duplicates.
   When per-change review was used, dedup within a change first, then across.
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

Present findings grouped by severity, then by change (in stack order), then by file.
Use this format:

```
## Code Review: <scope description>

Scope: `<revset>` (N changes, M files, K lines changed)

### P1 — Critical (N findings)

**<title>** — `path/to/file.py:42` (change `abcdefg`)
<description with concrete fix>

### P2 — Important (N findings)

...

### P3 — Suggestions (N findings)

...

### Summary
- N changes reviewed across <revset>, M findings (X P1, Y P2, Z P3)
- Key themes: <1-2 sentence synthesis of what the findings reveal>
- Dismissed (false positives): <count, briefly listed>
```

If there are zero findings at a severity level, omit that section entirely.
If there are zero findings total, say so clearly — don't invent issues to fill space.

## Phase 5: Post findings

Findings should be captured somewhere durable, not just displayed in-session. The
posting target depends on whether the changes are already on remote bookmarks with
PRs open.

### Determine the target

For each change in scope, check whether it has a bookmark and an open PR:

```bash
# For change abcdefg, find any bookmark pointing at it
jj log -r abcdefg --no-graph -T 'bookmarks ++ "\n"'
# For each bookmark, check for an open PR
gh pr list --head <bookmark-name> --state open --json number,url
```

This produces one of three states per change:
1. **Bookmark + PR exists** — post findings to that PR
2. **Bookmark exists, no PR** — bookmark is local-only or PR was closed
3. **No bookmark** — change hasn't been published yet

### Option A: Per-change posting (when reviewing a stack with PRs)

When the scope is a stack and changes have associated PRs, post each change's
findings as a comment on its corresponding PR:

```bash
gh pr comment <N> --body <findings-for-this-change>
```

Findings that span multiple changes (e.g., "the same gap appears in change 1 and 3")
get posted to the highest-priority change's PR with a note linking to the others.

### Option B: Single PR posting

When the scope is a single PR or single change with one PR, post the full review
as one comment.

### Option C: Tracking issue

If no PRs exist (work hasn't been published) but there's a tracking issue:
```bash
gh issue comment <N> --repo <repo> --body <review>
```

### Option D: Display in-session

If none of the above apply, displaying the review in the conversation is sufficient.
The findings are still valuable even without a durable destination.

Always tell the user where the review was posted (which PR(s), issue, or "displayed
in-session"), with a per-change breakdown when reviewing a stack.

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
- **Use change IDs, not commit IDs** — change IDs are stable across rebases. Findings that
  reference commit IDs become stale the first time you `jj squash` or `jj rebase`.
