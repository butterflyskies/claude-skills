---
name: land
description: End-of-session wrap-up. Lands outstanding stack work via /develop, syncs PR descriptions, updates project tracker, captures wins, writes handoff notes.
disable-model-invocation: false
---

# /land — End-of-Session Wrap-up

Run this when wrapping up a work session. Work through each phase in order.

This skill is jj-native. The heavy lifting of pushing changes and managing PRs lives
in `/develop`'s Phase 5, which `/land` invokes for any repos with unpushed stack
work. See [references/stacking-conventions.md](../references/stacking-conventions.md)
for the bookmark and PR mechanics.

Use memory-mcp's `recall` or `read` tool to load the `required-environment-variables` memory
if you haven't already this session, and use those identities for all jj/git/gh operations.

Use `recall` to find the `infrastructure-overview` memory for GitHub org conventions, project
tracker locations, and issue routing rules.

## Idempotency

This skill may be run more than once per session (e.g. after additional work). Each phase
should check whether there's actually something new to do before acting:
- Don't push a stack that's already pushed and unchanged
- Don't update a PR description that already matches the latest stack
- Don't duplicate tracker comments — check recent comments before posting
- Don't rewrite the flight-log entry — append new items or skip if nothing changed
- Don't overwrite handoff notes — merge new context into existing (per-project only)

## Phase 1: Capture learnings

Review what happened this session and update relevant memories via memory-mcp:
- New project knowledge, architecture changes, resolved design questions
- Workflow preferences or patterns that emerged
- Corrections to stale information (test counts, binary sizes, branch status, etc.)

Use memory-mcp tools (`edit` / `remember`) for all updates. Run `sync` after.

## Phase 2: Stack landing per repo

For each repo touched this session:

1. **Check for unpushed work.**
   ```bash
   cd <repo>
   jj log -r 'stack()'
   ```
   If `stack()` is empty (working copy is on a published bookmark with nothing on top),
   nothing to do for this repo — skip to the next.

2. **Run pre-land quality gate.** For Rust projects:
   ```bash
   cargo fmt -- --check && cargo clippy -- -D warnings && cargo nextest run --workspace
   ```
   If any check fails, fix the issue first. Fixes go in a new change at the top of the
   stack (do not amend earlier layers). If the fix belongs in an earlier layer, follow
   the squash-into-target-layer pattern from `/develop` Phase 4.5. For other languages,
   run the equivalent formatter, linter, and test suite.

3. **Invoke /develop land.** This handles bookmark setting, push, PR open/update, base
   fixups, and stack footer regeneration. The whole Phase 5 workflow from `/develop`.

   ```
   /develop land
   ```

   `/develop land` is idempotent — running it on a stack that's already pushed and
   has up-to-date PRs is a no-op (it just regenerates the stack footers, which is
   cheap). Running it after a stack rewrite moves bookmarks, force-pushes, and fixes
   PR bases.

4. **Record the result.** Note which PRs were created or updated for use in later phases.

Skip repos with empty stacks.

## Phase 3: PR description audit

For any open PRs related to this session's work that aren't part of an active stack
(e.g., a draft PR from earlier work, or a PR whose stack you've since merged the
bottom of):

1. List commits on the PR's bookmark
2. Compare against the current PR body
3. If the description is stale or incomplete, update it via `gh pr edit`

Stack footers are managed by `/develop` Phase 5f and don't need attention here. This
phase is for PRs *outside* an active stack.

## Phase 4: Flight Plan project tracker

1. List open items from the master project board
2. For items that progressed this session, comment on the issue with specifics —
   include PR links from Phase 2 where relevant
3. For new work that should be tracked, create issues following the routing rules in
   the `infrastructure_overview` memory
4. Move items to appropriate status columns if needed

## Phase 5: Notification triage

Use the **gh-notify MCP tools** to clean up notifications related to this session's work.

1. Call `sync_notifications` to refresh state from GitHub
2. Call `list_actionable` to see what's still `[NEW]` or `[TRIAGED]`
3. For notifications that correspond to PRs merged, issues closed, or work completed
   during this session: call `mark_acted` (this also marks them read on GitHub)
4. For CI noise (ci_activity on bookmarks that were re-pushed, cancelled runs, etc.):
   call `dismiss` with a brief reason

Only act on notifications clearly related to this session's work. Leave unrelated
notifications for the next `/briefing` to surface.

Skip this phase if gh-notify MCP tools are not available.

## Phase 6: Milestone rollover

Manage date-based weekly milestones in `butterflyskies/tasks`. The naming convention
is `Friday Focus — Mon DD` (e.g., "Friday Focus — Feb 13").

1. List all open milestones: `gh api repos/butterflyskies/tasks/milestones?state=open`
2. For any milestone whose `due_on` is in the past:
   a. If a newer milestone doesn't already exist, create one for the next Friday:
      - Title: `Friday Focus — <Mon DD>` (e.g., "Friday Focus — Feb 20")
      - Due date: the Saturday after that Friday at `T00:00:00Z` (so all of Friday is
        available before the milestone is considered past due)
      - Description: "Weekly milestone — open items carry forward automatically at session end."
   b. Move all open issues from the past-due milestone to the current/new milestone.
      The GitHub API requires `milestone` as a JSON integer, not a string. Use:
      ```
      gh api repos/<owner>/<repo>/issues/<num> --input - <<< '{"milestone": <int>}'
      ```
      Do NOT use `-f milestone=<num>` (sends a string, gets HTTP 422).
   c. After carry-forward, check the old milestone's issue counts:
      - If 0 open AND 0 closed → delete it (`DELETE /milestones/:id`)
      - Otherwise → close it (`PATCH state=closed`)
3. Ensure any new issues created during this session are assigned to the current milestone

Skip this phase entirely if no milestones are past due and the current milestone exists.

## Phase 7: Flight log

Write the day's wins entry in `butterflyskies/flight-log`:
1. Clone or pull the repo
2. Create/update `entries/YYYY-MM-DD.md` for today
3. Format: H1 date header, then bolded theme lines with bullet-point details underneath
4. Link to relevant PRs, issues, and commits where appropriate. For stacked work,
   link the whole stack ("PRs #123, #124, #125" or a stack-overview comment in the
   tracking issue) rather than just the bottom layer.
5. Commit and push (this happens in flight-log's repo using its own jj stack —
   typically a single change since flight-log entries don't stack)

If the entry already exists, append new items — don't rewrite what's there.

If a "so what?" framing was established at the start of the session (e.g. via `/develop`
Phase 0), write the entry against that intent — confirm what was achieved relative to
the original motivation, not just list what was done. The framing makes entries meaningful
when read back later.

The tone should be celebratory and specific — what was built, what was fixed, what was
figured out. Not a dry changelog; a record of progress that's satisfying to read back.

## Phase 8: Session handoff

Write a **project-scoped** handoff memory (e.g. `session-handoff` with `scope: "project:<name>"`)
via memory-mcp's `remember` tool, only when there is genuinely complex in-flight context that
would take time to re-derive.

When writing one:
- What was being worked on and where it left off
- The state of the stack (which layers are landed, which are in review, which are still local)
- Decisions that were in-flight or deferred
- Context that would take time to re-derive
- Anything the next session should pick up first

This is NOT a todo list (that's the tracker) — it's the thread of intent.

**Do not write a global-scope `session_handoff` memory.** Global handoffs race with
concurrent sessions across different projects — each landing session would overwrite
the other's context. Project-scoped handoffs are isolated and safe. When a project's
handoff is no longer relevant (work completed, context captured in issues/memories),
delete it.

Skip this phase if the session's state is fully captured by issue trackers, PR
descriptions, and project memories already written in earlier phases.
