---
name: design
description: "Structured design process for software projects. Produces specification artifacts (problem space, requirements, architecture, threat model, test plan) in docs/design/ before implementation begins. Scales from lightweight to full ceremony based on scope."
---

# /design — Structured Design Process

Produce design artifacts before implementation. The output lives in `docs/design/` and
becomes the source of truth that `/develop` works from.

Use memory-mcp to load `required-environment-variables` and any project-scoped memories
(use `list` filtered by project scope) if not already loaded this session.

## Argument handling

`$ARGUMENTS` describes what to design. It can be:

| Form | Meaning |
|------|---------|
| Free text | Describe the thing to design — run from Phase 0 |
| `problem <text>` | Phase 1 only — define the problem space |
| `requirements <text>` | Phases 1-2 — problem space through requirements |
| `architecture <text>` | Phases 1-3 — through architecture |
| `threat-model` | Phase 4 only — threat model existing architecture in docs/design/ |
| `test-plan` | Phase 5 only — derive test plan from existing SRTM |
| `review` | Re-read docs/design/, check effectiveness signals, suggest revisions |
| `<component>: <text>` | Component-scoped — artifacts go in docs/design/\<component\>/ |

When a specific phase is requested, check that prerequisite artifacts exist. If
`docs/design/architecture.md` doesn't exist and the user asks for `threat-model`,
say so and offer to run the earlier phases first.

## Coordinator responsibilities

You are the facilitator. Your job:
1. Parse the task and calibrate depth (Phase 0)
2. Guide the user through each phase via structured questions and proposals
3. Produce draft artifacts and present them for review
4. Dispatch sub-agents for mechanical generation (diagrams, threat enumeration)
5. Write artifacts to `docs/design/` after user approval at each gate
6. Maintain the design index (`docs/design/README.md`)

You do NOT: make design decisions unilaterally, skip human gates, or treat any
phase as mandatory without discussing scope with the user first.

## Phase 0: Scope and calibrate

Before starting the design process, establish how much ceremony this work needs.

Read the task description and any existing `docs/design/` artifacts. Also check for a global `design-effectiveness` memory via memory-mcp `recall` — if it
exists, read it. Previous patterns inform the depth recommendation (e.g., "lightweight
has been the right call for most designs so far" or "code review found security gaps
last time threat modeling was skipped").

Then assess:

- **Is this greenfield or extending something existing?**
  Greenfield needs more design. Extensions may only need incremental updates.
- **What's the blast radius?** A new internal utility vs. a public API vs. a
  security-critical system each warrant different depth.
- **Are there existing design artifacts?** If `docs/design/` exists, this may be
  an iteration rather than a fresh design.

Propose a depth to the user:

| Depth | When | Phases |
|-------|------|--------|
| **Lightweight** | Small feature, well-understood domain, low risk | 1 (abbreviated) -> 2 (key requirements only) -> skip to 5 |
| **Standard** | New component, moderate complexity, some unknowns | 1 -> 2 -> 3 -> discuss 4 -> 5 |
| **Full** | Greenfield system, security-critical, public API, regulatory | 1 -> 2 -> 3 -> 4 -> 5 |

State your recommendation and why. **Wait for the user to confirm or adjust
before proceeding.** The user may override in either direction.

## Phase 1: Problem space

This phase is collaborative — no sub-agent. Work through these questions with the
user, drafting as you go:

### What are we solving?
- What problem exists today? Who experiences it?
- What triggers the need for this solution now?

### Inputs and outputs
- What data or events enter the system?
- What does the system produce? For whom?
- What are the key transformations between input and output?

### Boundaries
- What is explicitly out of scope?
- What adjacent systems does this interact with?
- What constraints exist (technical, organizational, regulatory)?

### Success criteria
- How would we know this design succeeded?
- What would failure look like?

Draft the problem space document as you discuss. When the conversation converges:

**Artifact**: Write `docs/design/problem.md`
**Gate**: Present the draft to the user. **Always wait for explicit approval
before proceeding to Phase 2.**

## Phase 2: Concept development

Work through use cases and requirements with the user. This phase builds the
foundation that architecture and testing are derived from.

### 2a. Use cases

For each actor identified in Phase 1, enumerate:
- **Use cases**: what does this actor need to accomplish?
- **Abuse cases**: how could a malicious actor misuse this capability?
- **Security use cases**: what security behaviors must the system exhibit?
  (authentication, authorization, audit, data protection)

Present use cases in a structured table:

```
| ID | Actor | Use Case | Type | Priority |
|----|-------|----------|------|----------|
| UC-01 | User | Upload document for processing | Normal | Must |
| AC-01 | Attacker | Upload malicious payload | Abuse | Must-mitigate |
| SC-01 | System | Validate file type before processing | Security | Must |
```

### 2b. Requirements

Derive requirements from use cases. Each requirement should be:
- **Testable** — has clear pass/fail criteria
- **Traceable** — links back to one or more use cases

For requirements that touch security, authentication, session management, access
control, or data protection: reference the relevant OWASP ASVS category as a
"have we considered this?" prompt. The ASVS categories:

- V1: Architecture, design, threat modeling
- V2: Authentication
- V3: Session management
- V4: Access control
- V5: Validation, sanitization, encoding
- V6: Stored cryptography
- V7: Error handling, logging
- V8: Data protection
- V9: Communication
- V10: Malicious code (supply chain)
- V11: Business logic
- V12: Files and resources
- V13: API and web services
- V14: Configuration

Do NOT apply all categories mechanically. Flag the ones relevant to this project's
domain and ask the user which merit deeper analysis. Record which categories were
reviewed and which were deemed not applicable, with a one-line rationale.

### 2c. Security Requirements Traceability Matrix (SRTM)

Build a traceability matrix linking:
`Use Case -> Requirement -> ASVS Category (if applicable) -> Test Case (placeholder)`

The test case column starts as placeholders (e.g., "TC-01: verify...") — Phase 5
fills these in as a concrete test plan.

```
| Req ID | Requirement | Source UC | ASVS | Test Case |
|--------|-------------|-----------|------|-----------|
| R-01 | System shall validate file type | UC-01, AC-01 | V12.1 | TC-01 (pending) |
```

**Artifact**: Write `docs/design/requirements.md` (includes use case table, requirements,
ASVS review notes, and SRTM)
**Gate**: Present to user. **Always wait for explicit approval before proceeding
to Phase 3.**

## Phase 3: Architecture

Design the system architecture. This phase uses a sub-agent for diagram generation
after the user approves the architectural decisions.

### 3a. Architectural decisions (coordinator + user)

Work through these with the user collaboratively:
- **Component decomposition**: what are the major components and their responsibilities?
- **Data model**: what are the key entities and relationships?
- **Integration points**: how do components communicate? What protocols?
- **Technology choices**: languages, frameworks, infrastructure — and why?

For each significant decision, note it for an ADR (written during `/develop` Phase 1.5,
or written here if the user prefers — ask).

### 3b. Diagram generation (sub-agent)

After architectural decisions are agreed, dispatch an **architecture sub-agent**
(model: sonnet) to generate Mermaid diagrams.

**Sub-agent prompt template:**
```
You are a technical documentation agent. Generate Mermaid diagrams based on
the architectural decisions provided. Produce clean, readable diagrams — not
exhaustive detail.

Architectural decisions:
<decisions from 3a>

Requirements:
<from docs/design/requirements.md>

Generate these diagrams in Mermaid syntax:

1. **System context diagram** — the system as a box, external actors and systems
   around it, showing data flows. Use a C4-style approach.

2. **Component diagram** — internal components, their responsibilities, and
   how they communicate. Include data stores.

3. **Data flow diagram with trust boundaries** — show where data crosses trust
   boundaries (user <-> API, API <-> database, internal <-> external service).
   Mark trust boundaries explicitly with subgraph labels.
   This diagram is the primary input for threat modeling in Phase 4.

4. **Data schema** — entity-relationship diagram for the core data model.
   Include key fields only, not every column.

5. **Key sequence diagrams** — for the 2-3 most important or complex
   interactions identified in the use cases. Not every use case needs one.

For each diagram, include a brief prose description (2-3 sentences) explaining
what the diagram shows and any notable design choices visible in it.

Output format: markdown with ```mermaid code blocks, each preceded by an H3
heading and the prose description.
```

### 3c. Review and iterate

Present the generated diagrams to the user. Diagrams frequently need iteration —
components may be misnamed, flows may be wrong, trust boundaries may be misplaced.

**Critical:** AI-generated architecture diagrams often contain subtle but significant
flaws — missing trust boundaries, inappropriate data flows, wrong component
responsibilities. The human review gate here is load-bearing. Your job: present the
diagrams, flag anything you're uncertain about, and iterate until the user is satisfied.

Re-dispatch the sub-agent for significant changes; make minor edits directly.

**Artifact**: Write `docs/design/architecture.md` (prose decisions + Mermaid diagrams)
**Gate**: Present final architecture to user. **Always wait for explicit approval
before proceeding to Phase 4.**

## Phase 4: Threat model (gated)

This phase is the heaviest. Before starting, have an explicit conversation with the
user about whether to proceed.

### Gate discussion

Present this to the user:

> **Threat modeling checkpoint.**
>
> Based on the architecture, I've identified these trust boundaries and data flows:
> - [list trust boundaries from the data flow diagram]
> - [list external-facing data flows]
>
> A STRIDE analysis would systematically evaluate each data flow crossing a trust
> boundary for: Spoofing, Tampering, Repudiation, Information Disclosure, Denial
> of Service, and Elevation of Privilege.
>
> This is the most time-intensive phase of the design process. It's most valuable for:
> - Systems with external-facing APIs or user authentication
> - Systems that handle sensitive data
> - Systems where a security incident would have significant impact
>
> Options:
> 1. **Proceed with full STRIDE analysis** — thorough, takes time
> 2. **Lightweight review** — I'll flag the most obvious concerns without
>    systematic enumeration
> 3. **Defer** — capture the trust boundaries now, do the analysis later
>    (you can run `/design threat-model` when ready)
> 4. **Skip** — not needed for this scope

If a `design-effectiveness` memory exists and contains relevant signals (e.g.,
"code review found security gaps last time threat modeling was skipped"), mention
this in the discussion — it's a data point, not a mandate.

**Wait for the user's choice.** Do not default to any option.

### If proceeding (option 1 or 2):

Dispatch a **threat modeling sub-agent** (model: opus) to analyze the architecture.

**Sub-agent prompt template:**
```
You are a threat modeling agent performing STRIDE analysis. You are methodical
and precise. You do not invent threats that don't apply — false positives waste
the engineer's time and erode trust in the process. You do find threats that a
developer might overlook.

Architecture:
<from docs/design/architecture.md — include all diagrams>

Requirements:
<from docs/design/requirements.md>

Data flow diagram with trust boundaries:
<extract specifically from architecture.md>

For each data flow that crosses a trust boundary, analyze:

| Threat | Question |
|--------|----------|
| **Spoofing** | Can an entity be impersonated on this flow? |
| **Tampering** | Can data be modified in transit or at rest? |
| **Repudiation** | Can actions on this flow occur without accountability? |
| **Information Disclosure** | Can data leak via this flow (logs, errors, side channels)? |
| **Denial of Service** | Can this flow be used to exhaust resources? |
| **Elevation of Privilege** | Can this flow be used to gain unauthorized access? |

For each identified threat:
1. Describe the threat concretely (not "tampering is possible" but "an attacker
   could modify the JWT payload because...")
2. Assess likelihood (low/medium/high) and impact (low/medium/high)
3. Propose a mitigation — either a new requirement or an architectural change
4. If the mitigation is a new requirement, format it as: "NEW-REQ: <requirement text>"
   with a suggested ASVS category

Also check for:
- Injection vectors at each input boundary
- Authentication/authorization gaps in the flow
- Sensitive data exposure in logs, errors, or API responses
- Resource exhaustion vectors (unbounded queues, connections, memory)
- Supply chain concerns (dependencies with excessive privilege)

Output format:

## Trust Boundary: <name>
### Data Flow: <source -> destination>
| STRIDE | Threat | Likelihood | Impact | Mitigation |
|--------|--------|------------|--------|------------|
| S | ... | ... | ... | ... |

## New Requirements from Threat Model
- NEW-REQ-01: <text> (ASVS: <category>)
- ...

## Architectural Changes Recommended
- <change and rationale>
```

### Iteration back to requirements and architecture

After the threat model sub-agent returns, present findings to the user. Then:

1. **New requirements** (NEW-REQ items): add to `docs/design/requirements.md` and
   update the SRTM. These get traced like any other requirement.
2. **Architectural changes**: update `docs/design/architecture.md`. Re-generate
   affected diagrams if needed.
3. If changes are significant, discuss whether another threat model pass is needed
   on the updated architecture. One iteration is usually sufficient; diminishing
   returns set in quickly.

This iteration — where threat modeling surfaces issues that change requirements and
architecture — is where real engineering happens. It's the process working, not failing.

**Artifact**: Write `docs/design/threat-model.md`
**Gate**: Present threat model and any resulting requirement/architecture changes
to user. **Always wait for explicit approval before proceeding to Phase 5.**

## Phase 5: Verification plan

Derive a test strategy from the SRTM. This phase bridges design into implementation —
the output is what `/develop`'s planning agent uses to ensure tests are written.

For each requirement in the SRTM:
1. Define a concrete test case (replacing the placeholder from Phase 2)
2. Classify as: unit, integration, or system test
3. Identify what needs to be true for the test to be meaningful
   (test fixtures, mocks, environment)

For requirements derived from threat modeling (NEW-REQ items):
- These often map to security tests (e.g., "verify that expired JWTs are rejected")
- Note which can be automated vs. which need manual review

```
| Test ID | Requirement | Type | Description | Automated? |
|---------|-------------|------|-------------|------------|
| TC-01 | R-01 | Unit | Upload handler rejects non-whitelisted MIME types | Yes |
| TC-02 | NEW-REQ-01 | Integration | Expired JWT returns 401, not partial data | Yes |
```

**Artifact**: Write `docs/design/test-plan.md`
**Artifact**: Update `docs/design/requirements.md` SRTM with final test case IDs
**Artifact**: Write/update `docs/design/README.md` — index linking all artifacts
with a brief status note for each

**Gate**: Present the complete design package to the user. Confirm it's ready to
hand off to `/develop`.

## Review mode

When invoked with `review`, this mode synthesizes effectiveness signals and checks
design health. No sub-agent needed.

1. **Read existing artifacts**: scan `docs/design/` for all files, check metadata
   status and last-updated dates
2. **Check for drift**: compare design artifacts against current code via git log.
   If significant implementation has happened since the design was last updated,
   flag potential drift.
3. **Read effectiveness memory**: use memory-mcp `recall` for `design-effectiveness`
   scoped to this project. If observations exist, summarize patterns.
4. **Cross-reference review findings**: if `code-review-patterns` memory exists,
   check whether security findings in this project map to gaps in the design
   (threat classes not covered, trust boundaries not enumerated).
5. **Check test coverage against SRTM**: if the test plan exists, compare SRTM
   test cases against actual test files. Tests that exist but aren't in the SRTM
   indicate organic discovery — the design missed something worth backfilling.
6. **Present findings**: summarize what's working, what's drifted, what's missing.
   Suggest concrete revisions — both "add coverage here" and "this section isn't
   pulling its weight, consider simplifying."

## Artifact format

All artifacts live in `docs/design/` (or `docs/design/<component>/` for scoped designs).

### File structure
```
docs/design/
  README.md          — index, status, scope summary
  problem.md         — problem space definition
  requirements.md    — use cases, requirements, SRTM, ASVS notes
  architecture.md    — decisions, Mermaid diagrams
  threat-model.md    — STRIDE analysis (if performed)
  test-plan.md       — verification strategy from SRTM
```

### Conventions
- **Mermaid for all diagrams** — renders in GitHub, VS Code, and most doc tools
- **Cross-references use relative links** — `[requirements](requirements.md)`
- **Each file starts with a metadata comment:**
  ```markdown
  <!-- design-meta
  status: draft | review | approved
  last-updated: YYYY-MM-DD
  phase: 1 | 2 | 3 | 4 | 5
  -->
  ```
- **README.md** is the entry point — it links to all other artifacts and notes
  which phases have been completed, which were skipped, and why

### Connection to /develop

When `/develop` starts, its planning agent should check for `docs/design/README.md`.
If it exists:
- Use the requirements and architecture as the plan's foundation
- Reference specific requirement IDs when describing what to implement
- Use the test plan to ensure test coverage maps to the SRTM
- Flag any plan decisions that conflict with the design artifacts

(This is a follow-on edit to `/develop` — the `/design` skill writes artifacts in
a format that makes this discovery straightforward.)

## Effectiveness tracking

This skill participates in a distributed feedback loop across the skill system.
The goal: automatically track whether design work is utilized effectively and surface
signals for revision.

### What this skill contributes

- **Phase 0**: reads `design-effectiveness` memory if it exists. Previous patterns
  inform the depth recommendation.
- **Phase 4 gate**: mentions relevant effectiveness signals (e.g., past threat model
  gaps found by code review) as data points in the discussion.
- **Review mode**: synthesizes all accumulated observations and suggests revisions.

### What other skills contribute (follow-on changes)

- **`/code-review` Phase 6**: when `docs/design/` exists, notes whether security
  findings map to threat model coverage or represent gaps. Appends to
  `design-effectiveness` memory.
- **`/land` Phase 1**: when design artifacts exist, notes whether they were referenced
  during the session. Appends.
- **`/develop` Phase 1**: notes whether design artifacts informed the plan. Appends.

### Memory format

Global memory `design-effectiveness` (scope: `global`). This tracks how the design
process itself is working across all projects — it's about the methodology, not any
single project's design. Each observation is a dated one-liner with project context:

```
2026-04-15 [elfin]: /code-review found auth bypass (P1) — threat model covered auth
flows but missed the /admin path. Gap in trust boundary enumeration.
2026-04-18 [memory-mcp]: /develop planning agent used requirements.md as plan
foundation — utilization confirmed.
2026-04-22: Phase 0 chose "lightweight" for the third consecutive design — full
process may be over-specified for typical task size.
```

## Model selection

| Sub-agent | Model | Rationale |
|-----------|-------|-----------|
| Architecture diagrams (Phase 3b) | **sonnet** | Mechanical generation from agreed decisions |
| Threat modeling (Phase 4) | **opus** | Judgment-heavy — must distinguish real threats from noise |

Phases 0, 1, 2, and 5 are coordinator-led (collaborative with the user). The
coordinator inherits the session model. No sub-agents needed — the value is in the
conversation, not parallel analysis.

## Guidelines

- **The process serves the project, not the other way around.** If a phase isn't
  adding value for this particular scope, skip it — but record that you skipped it
  and why.
- **Draft, don't polish.** First-pass artifacts should be good enough to reason from,
  not publication-ready. Iteration refines them.
- **Diagrams are communication tools, not specifications.** A diagram that's accurate
  but unreadable has failed. Prefer clarity over completeness.
- **Threat modeling finds threats, not solutions.** The threat model identifies what
  could go wrong. Mitigations are proposed as new requirements that go through the
  normal prioritization process — not all threats need immediate mitigation.
- **ASVS is a prompt, not a mandate.** Use it to ask "have we thought about this?"
  not as a compliance checklist. Record which categories were considered and which
  were explicitly set aside.
- **Iteration is the point.** Threat modeling that surfaces new requirements which
  change the architecture which needs re-evaluation — that's the process working.
  But one iteration loop is usually sufficient; diminishing returns are real.
- **Small scope, small ceremony.** Phase 0 calibration prevents over-engineering the
  process itself. A design for "add a CLI flag" should take 5 minutes, not 5 phases.
- **Track what works.** Effectiveness signals from reviews and implementation inform
  future calibration. The process should get better every time it's used.
