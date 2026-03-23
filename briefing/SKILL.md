---
name: briefing
description: "When requested: session-start situational awareness — notifications, PRs, tasks, and due follow-ups."
---

# /briefing — Situational Awareness

Scan what's happened since last session and present a concise summary. This skill is
**read-only** — it never modifies anything (except syncing notification state via gh-notify).

Use memory-mcp's `read` tool to load the `required-environment-variables` memory (scope: global)
if you haven't already this session, and use those identities for all gh operations throughout.

## Argument handling

`$ARGUMENTS` is optional. If provided, run only the matching phase(s):

| Keyword          | Phase(s) run |
|------------------|-------------|
| `notifications`  | 1 only |
| `prs`            | 2 only |
| `tasks`          | 3 only |
| `followups`      | 4 only |
| *(empty)*        | All four |

Multiple keywords can be combined (e.g. `prs tasks`). Match case-insensitively.

## Phase 1: Notifications

Use the **gh-notify MCP tools** (not raw `gh api`):

1. Call `sync_notifications` to fetch and upsert from GitHub
2. Call `list_actionable` to get only `[NEW]` and `[TRIAGED]` items
3. Call `get_stats` for the summary counts

This skips already-acted/dismissed notifications from previous sessions.

- Group actionable items by `reason` (mention, author, comment, ci_activity, assign, etc.)
- Under each reason heading, list: repo name + title, with status tag (`[NEW]`/`[TRIAGED]`)
- If no actionable notifications, collapse to: **Notifications: none**
- After presenting, offer to triage: "Want me to dismiss CI noise or triage any of these?"

## Phase 2: Open PRs

Check PRs involving both identities:

```bash
gh search prs --state=open --involves=butterflysky --json repository,title,number,url,updatedAt
gh search prs --state=open --involves=butterflysky-ai --json repository,title,number,url,updatedAt
```

- Deduplicate by URL
- For each PR, check CI status (note: `gh pr checks` does not support `--json`):
  ```bash
  gh pr checks <number> --repo <owner/repo>
  ```
- For each PR, check for recent review comments (last 7 days):
  ```bash
  gh api "repos/<owner>/<repo>/pulls/<number>/reviews" --jq '[.[] | select(.submitted_at > "<7_days_ago_ISO>") | {user: .user.login, state: .state, body: .body}]'
  gh api "repos/<owner>/<repo>/pulls/<number>/comments" --jq '[.[] | select(.updated_at > "<7_days_ago_ISO>") | {user: .user.login, body: .body, path: .path, created_at: .created_at}]'
  ```
- Surface unresolved review comments in the PR table output — show reviewer name, comment count, and whether changes were requested
- If no open PRs, collapse to: **Open PRs: none**

## Phase 3: Tasks & Milestones

From the project tracker. Note: quote the milestones URL to prevent zsh glob expansion on `?`:

```bash
gh issue list --repo butterflyskies/tasks --state open --json number,title,milestone,labels,assignees
gh api 'repos/butterflyskies/tasks/milestones?state=open'
```

- Group open issues by milestone
- Show milestone title, due date, and open/closed counts
- List unassigned-to-milestone issues separately
- Highlight anything past due
- If no open tasks, collapse to: **Tasks: none**

## Phase 4: Follow-ups due

Use memory-mcp's `recall` tool to search for periodic follow-ups. If a `periodic-followups`
memory exists (scope: global), read it. For each active item:

1. Parse `Last checked` date and `Frequency`
2. Calculate next due date:
   - weekly = last checked + 7 days
   - biweekly = last checked + 14 days
   - monthly = last checked + 30 days
3. Compare against today's date

Report due/not-yet-due status. Point users to `/check-in` to execute the actual checks.

**Do not execute follow-up checks** — that's `/check-in`'s job.

## Output format

### Notifications

Show stats summary line first, then group actionable items by reason with status tags.
Condense related items from the same repo when there are many:

```
## Notifications (12 actionable / 30 total — 18 previously acted/dismissed)

**mention** (2)
- [NEW] `oraios/serena` — Add global memories support (PR)
- [TRIAGED] `butterflyskies/cc-toolgate` — v4: tree-sitter-bash AST (PR)

**author** (5)
- [NEW] `butterflyskies/gossamer` — Add connect() syscall tracer (PR)
- [NEW] `butterflyskies/elgato-control-home-automation` — 3 PRs (remove hardcoded lights, ...)
- ...

**ci_activity** (3)
- ...

> Dismiss CI noise or triage any of these? (use thread IDs from list_actionable)
```

### Open PRs

Markdown table with linked PR number, title, repo, CI summary, review status, and last-updated date.
If a PR has unresolved review comments, add a "Reviews" row beneath it listing reviewer, state, and
comment count:

```
## Open PRs

| PR | Repo | CI | Reviews | Updated |
|----|------|----|---------|---------|
| [#1007](url) Add global memories support | oraios/serena | all pass | — | Feb 19 |
| [#16](url) Add system event parser | butterflyskies/elfin | all pass | codex: changes_requested (3 comments) | Mar 5 |
| [#9](url) Track Discord disconnect events | butterflysky/argus | pass | — | Jan 6 |
```

For PRs with review comments, briefly list the key concerns below the table under a
`### Review comments needing attention` subheading, grouped by PR.

### Tasks & Milestones

Group by milestone, showing due date and counts, then list issue numbers and titles.
Unassigned issues in a separate group:

```
## Tasks & Milestones

**Friday Focus — Feb 27** (due Feb 26, 14 open / 0 closed)
- #69 Design adversarial code review sub-agent for cc-toolgate
- #68 Gossamer — system observability toolbox
- ...

**Unassigned to milestone** (N issues)
- #70 Track upstream Serena memory issues
- ...
```

### Follow-ups

Per-item status with last-checked date and next-due date:

```
## Follow-ups

**Serena PR #1007 — Global Memories** (weekly)
- Last checked: 2026-02-19
- Next due: 2026-02-26
- Not yet due. Use `/check-in 1` when ready.
```

### General rules

- Empty sections get a single "none" line, not omitted entirely
- Keep it scannable — no prose paragraphs, just structured lists and tables
