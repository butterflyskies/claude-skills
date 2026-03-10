---
name: publish
description: "Publish skill changes — detects edits, creates a feature branch, commits, pushes, and opens a PR against main. Invoke after modifying any skill files."
---

# /publish — Publish Skill Changes

Handles the git workflow for skill edits so any session can modify skills and land them
properly without manual git choreography.

Read the `required_environment_variables` Serena global memory if you haven't already this
session, and use those identities for all git/gh operations throughout.

## Prerequisites

This skill operates on the skills repo at `~/.claude/skills`. It assumes:
- The repo is initialized and has `origin` pointing to GitHub
- Branch protection on `main` requires PRs (no direct push)

## Phase 1: Detect and analyze changes

1. `git status` in `~/.claude/skills`
2. If the working tree is clean — report "Nothing to publish" and stop
3. `git diff` to review all changes (staged and unstaged)
4. If currently on a feature branch with unpushed commits, include those too

Review every changed file and understand what each edit does.

## Phase 2: Group changes into logical units

Analyze the changes and decide how to split them:

- **One logical change** (e.g. a single skill edited, or tightly related edits across files):
  one branch, one commit, one PR.
- **Multiple independent changes** (e.g. edits to unrelated skills, a new skill plus a
  separate fix to an existing one): separate branches, separate commits, separate PRs.

Guiding principles:
- Each PR should be reviewable on its own — a coherent, self-contained change
- Edits to the same skill that serve the same purpose belong together
- A new skill is its own PR unless it was created alongside tightly coupled edits to
  an existing skill
- When in doubt, split — smaller PRs are easier to review and merge

Summarize the grouping before proceeding (e.g. "I see two independent changes: X and Y.
I'll create separate PRs for each.").

## Phase 3: For each logical group, branch → commit → push → PR

Process each group sequentially. For each one:

### 3a. Ensure on the right branch
- Start from `main` (pull latest first)
- Create a new branch named after the change (e.g. `skill/develop-add-phase`,
  `skill/publish-new-skill`). Use `skill/` prefix.
- If already on a `skill/*` branch that matches this group: stay on it

### 3b. Commit
- Stage only the files belonging to this logical group
- Write a descriptive commit message summarizing the change
- Multiple commits within a group are fine if they represent distinct steps
  (e.g. "add new skill" then "wire up reference doc")

### 3c. Push
- Push to origin with `-u`

### 3d. Open or update PR
- If no PR exists for this branch: create one via `gh pr create`
  - Title: concise description of the skill change
  - Body: summary of what changed and why, with the standard footer
- If a PR already exists: update the description if new commits were added,
  or just report the PR URL

### 3e. Return to main
- `git checkout main` before starting the next group

## Phase 4: Wait for merge

After all PRs are created, report the PR URLs and **wait for the user** to confirm
each PR is merged. Do not switch back to main or pull until the user says the PR(s)
are merged.

## Phase 5: Return to main

Once the user confirms the PR(s) are merged:
1. `git checkout main && git pull`
2. Confirm the working tree is clean and up to date

## Output format

After creating PRs:
```
## Published skill changes

### PR 1: <title>
**Branch**: skill/<name>
**PR**: <url>
**Changes**:
- <bullet summary>

### PR 2: <title>  (if applicable)
...

Let me know when merged and I'll switch back to main.
```

After user confirms merge:
```
Switched to main, up to date.
```
