# Git Workflow: Stacked Diffs and Worktrees

Multiple Claude Code sessions may run concurrently against the same repo — from
parallel terminal windows, scheduled routines, or background agents. Sessions share
no coordination layer and have no reliable way to discover each other at runtime.
The working directory and index are unprotected shared state.

**Default: every session operates in its own worktree.** This is not an optimization —
it's a correctness requirement. Without worktrees, concurrent sessions will corrupt
each other's staging area and working directory silently.

When git town is available, use stacked branches to keep diffs small and reviewable.

## 1. Detection

Run at the start of Phase 0 alongside language detection:

```bash
# Check git town availability
if command -v git-town &>/dev/null; then
  GT_AVAILABLE=true
  # Check if configured for this repo
  if [ -f git-town.toml ] || [ -f .git-town.toml ] || [ -f .git-branches.toml ] || \
     [ -n "$(git config git-town.main-branch 2>/dev/null)" ]; then
    GT_CONFIGURED=true
  else
    GT_CONFIGURED=false
  fi
  # Check for suspended commands (must resolve before proceeding)
  PENDING=$(git-town status --pending 2>/dev/null || echo "")
  if [ -n "$PENDING" ]; then
    echo "WARNING: Suspended git town command: $PENDING"
  fi
else
  GT_AVAILABLE=false
fi
```

If git town is available but not configured, offer to create a minimal config:

```toml
# git-town.toml (repo root)
[branches]
main = "main"

[create]
share-new-branches = "push"

[propose]
breadcrumb = "stacks"

[sync]
feature-strategy = "rebase"
```

Report git town status to the user alongside language detection. If there's a
suspended command (`git-town status --pending`), it must be resolved first —
either `git town continue` (if the user wants to proceed) or `git town undo`.

## 2. Sub-agent interactivity rules

Claude Code permission prompts reach the user regardless of agent depth — the
harness routes them. But external tools that read from stdin, open a browser,
or invoke `$EDITOR` will block indefinitely in a sub-agent's non-interactive shell.

| Layer | Coordinator | Sub-agent |
|-------|:-----------:|:---------:|
| Claude Code permission prompts | OK | OK (harness routes to user) |
| External tool stdin prompts | OK (user present) | BLOCKS — no tty |
| Browser opens | OK | BLOCKS — no display |
| Editor invocations (`$EDITOR`) | Avoid | BLOCKS |

**Git town commands by interactivity:**

| Safe in sub-agents | Coordinator only (interactive) |
|--------------------|-------------------------------|
| `git town hack <name>` | `git town switch` (TUI picker) |
| `git town append <name>` | `git town init` (setup wizard) |
| `git town prepend <name>` | `git town up` (interactive when multiple children) |
| `git town sync [--stack]` | `git town propose` (may open browser) |
| `git town compress [--stack]` | |
| `git town branch` (display only) | |
| `git town diff-parent` | |
| `git town commit --down` | |
| `git town status` | |

Sub-agents should set `TERM=dumb` to suppress any residual TUI behavior.

For PR creation in sub-agents, use `gh pr create` directly instead of
`git town propose` — it's fully non-interactive and gives explicit control
over title, body, and base branch.

**Conflict handling**: When `git town sync` hits a merge conflict, it halts
and waits for `git town continue`. Sub-agents should detect this (non-zero
exit code), capture the conflict details, and return to the coordinator with
enough context for the user to decide how to proceed. Do not attempt automatic
conflict resolution in sub-agents.

## 3. Stacked diff workflow

Use stacking when changes are logically separable AND the combined diff would
exceed ~300 lines. Don't stack single-concern changes that happen to touch
multiple files — that's a normal branch.

### When to stack

- Refactor + feature (refactor first, feature builds on it)
- Multi-step migration (each step independently reviewable)
- Large feature with natural layers (data model → business logic → API)
- Review feedback on an earlier branch while continuing work on a later one

### Creating a stack

```bash
# Start from main — first branch in the stack
git town hack 1-refactor-module
# ... make changes, commit ...

# Stack the next branch on top
git town append 2-add-feature
# ... make changes, commit ...

# Optional: insert a branch between two existing ones
git town prepend 1b-extract-interface
```

Git town tracks the parent-child lineage automatically. `git town branch`
displays the full hierarchy.

### Committing into an ancestor branch

If you realize a change belongs in an earlier branch while working on a later one:

```bash
# Stage the relevant files (on the later branch)
git add <files-that-belong-earlier>
git town commit --down -m "fix: also handle edge case in refactor"
# This commits into the parent branch and syncs back into the current branch
# Use --down=2 for grandparent, etc.
```

### Syncing the stack

After any changes to an earlier branch (direct commits, merged PRs, rebases):

```bash
git town sync --stack    # propagate changes through all branches in the stack
```

### Creating PRs for the stack

**Coordinator** (has user present, browser OK):
```bash
git town propose --stack
# Creates a PR for every branch that doesn't have one yet
# Base branches are set correctly (each PR targets its parent branch)
# Embeds stack breadcrumbs in PR descriptions (if configured)
```

**Sub-agent** (non-interactive):
```bash
# Must create PRs individually with gh
# For each branch in the stack, set --base to the parent branch
gh pr create --head 1-refactor --base main --title "..." --body "..."
gh pr create --head 2-add-feature --base 1-refactor --title "..." --body "..."
```

### After a PR merges

```bash
git town sync --all
# Detects merged branches, deletes them locally, re-parents children onto main
# GitHub/GitLab auto-retargets the child PR to main when the parent branch is deleted
```

### Stack navigation

```bash
git town up       # move to child branch (interactive if multiple children)
git town down     # move to parent branch
git town branch   # display hierarchy
git checkout <branch-name>   # always safe, non-interactive
```

## 4. Worktree patterns

### Why worktrees are the default

Multiple Claude Code sessions can run concurrently against the same repo.
Sessions have no IPC, no shared process registry, and no way to discover
each other at runtime. The available cross-session mechanisms are all
pull-based and async:

| Mechanism | What it does | Limitation |
|-----------|-------------|------------|
| memory-mcp | `remember` / `recall` | Must explicitly poll — no push |
| Git refs | Commits visible across worktrees | No signaling — must check |
| GitHub PRs/issues | Durable async coordination | High latency |
| Filesystem markers | Lock files, sentinel files | No cleanup guarantee on crash |

Because no session can know whether another session is active in the same
repo, **every session that touches the working directory must operate in its
own worktree**. This applies to coordinators and sub-agents alike.

### Shared vs. isolated state

| Shared (all worktrees) | Isolated (per-worktree) |
|------------------------|------------------------|
| Object store (commits, blobs) | Working directory |
| Branch refs, tags, remotes | Index (staging area) |
| Stash (**never use in parallel!**) | HEAD |
| Config, hooks | Untracked files |

Branch refs are shared — a commit made in one worktree is instantly visible
from all others. This is what makes stacked branch workflows possible across
worktrees.

**Critical**: Stash is shared across worktrees. Concurrent stash operations
from parallel sessions will corrupt each other. Always commit, even as WIP.
Claude Code's `isolation: "worktree"` handles this correctly.

### Claude Code's built-in worktree support

The Agent tool accepts `isolation: "worktree"`, which:
- Creates a temporary git worktree for the sub-agent
- Gives it an isolated working directory and index
- Auto-cleans up if the sub-agent makes no changes
- Returns the worktree path and branch if changes were made

Use this for implementation and quality sub-agents. It handles the creation
and cleanup lifecycle automatically.

### Manual worktree management

For cases where you need explicit control (e.g., the coordinator itself
needs a worktree, or fixing review feedback on a stacked branch):

```bash
# Create worktree for an existing branch
git worktree add ../wt-feature-a feature-a

# Work in it
cd ../wt-feature-a
# ... make fixes, commit, push ...

# Clean up
git worktree remove ../wt-feature-a
```

A branch can only be checked out in one worktree at a time. Attempting to
check out an already-checked-out branch will fail.

### Build artifacts in worktrees

Each worktree has its own working directory, so:
- `node_modules`, `target/`, `.next/`, etc. are per-worktree
- Dependencies must be installed separately in each worktree
- For short-lived worktrees (edit → commit → push), skip dependency install

## 5. Git town + worktree interaction

### The sync gotcha (git-town/git-town#6083)

`git town sync` fails to detect shipped (merged) branches if they're still
checked out in a worktree. The sync silently does nothing instead of
re-parenting children.

**Required cleanup order:**
```bash
# 1. Remove worktrees for merged branches FIRST
git worktree remove ../wt-feature-a

# 2. Delete the local branch
git branch -d feature-a

# 3. THEN sync — now git town correctly detects the ship and re-parents
git town sync --all
```

If an agent creates worktrees for stack branches, it must clean them up
before calling `git town sync`. Establish this as a post-merge checklist.

### Combining stacks and worktrees for review fixes

Scenario: You have a stack `main → A → B → C`, you're on C, and review
feedback arrives on A.

```bash
# Option 1: Dispatch a sub-agent with worktree isolation
# The sub-agent checks out branch A in a worktree, fixes, commits, pushes
# Then the coordinator syncs the stack:
git town sync --stack   # propagates A's changes through B and C

# Option 2: Use git town commit --down (no worktree needed)
# If the fix is small and you have the changes staged:
git add <files>
git town commit --down=2 -m "fix: address review feedback on A"
# This commits into A (2 levels up) and syncs back into C
```

Option 1 is better for substantial fixes. Option 2 is better for one-liners.

## 6. Decision framework

### Worktrees (isolation)

Worktrees are the unconditional default. Don't gate on "is another session
active?" — you can't know.

```
Am I a sub-agent that writes files?
→ Use isolation: "worktree" on the Agent tool (automatic lifecycle)

Am I a coordinator session?
→ Create a worktree for your working branch at session start
→ Clean it up at session end (or via /land)

Only exception: single-session, single-repo, no background agents
→ Main worktree is safe (but worktree is still fine — low cost)
```

Cross-session coordination (knowing what other sessions are doing) is not
currently possible. Sessions have no stable identity, no IPC, and no push
mechanism. Worktrees eliminate the need for coordination at the working
directory level. When a proper channel-based communication layer exists,
revisit.

**Branch ownership**: worktrees isolate the working directory and index,
but branch refs are shared. Two agents committing to the same branch will
conflict on push (non-fast-forward rejection). This is the correct failure
mode — loud and recoverable. Do not force-push to resolve; return to the
user with the conflict.

The rule: **one agent owns a branch at a time.** Parallelize freely when
agents target different branches; serialize when they target the same branch.

| Scenario | Parallel? | Why |
|----------|:---------:|-----|
| Multiple read-only analyzers | Yes | No writes, no conflict |
| Write agents on different branches (each in own worktree) | Yes | Branch isolation |
| Write agents on the same branch | No | Concurrent commits conflict |
| Implementation on branch A + read-only quality check on A | Yes | Quality reads committed state |

This enables patterns like: fixing review feedback on branches A, B, and C
simultaneously with three parallel sub-agents in three worktrees, then
syncing the stack once all return. Across sessions, branch ownership is a
human coordination decision that the tooling cannot currently enforce.

### Stacking (branch organization)

```
Is git town available and configured?
├── YES: Does the task produce logically separable changes > ~300 LOC total?
│   ├── YES → Stack branches with git town
│   └── NO  → Single feature branch (git town hack)
└── NO: Create feature branches manually (git checkout -b)

Need to fix an earlier branch in a stack?
├── Small fix, changes already staged → git town commit --down
└── Substantial fix → Sub-agent with worktree isolation, then sync stack

Multiple independent changes to ship as separate PRs?
├── Git town available → Separate stacks (git town hack for each)
└── No git town → Sub-agents with worktree isolation, one per PR
```

## 7. Environment setup for sub-agents

When dispatching sub-agents that will run git town commands:

```bash
export TERM=dumb                          # suppress TUI dialogs
export GIT_TOWN_GITHUB_CONNECTOR=gh       # use gh CLI for API
export GIT_TOWN_SHARE_NEW_BRANCHES=push   # push branches immediately
```

Pass these as part of the sub-agent's environment context, alongside the
standard `GIT_CONFIG_GLOBAL` and `GH_CONFIG_DIR` variables from the
`required-environment-variables` memory.
