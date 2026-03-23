---
name: edit-skill
description: "Edit and publish skill changes. Takes a description of what to change as arguments, makes the edits, then handles the git workflow (branch, commit, push, PR). Use when you need to modify an existing skill or create a new one."
---

# /edit-skill — Edit and Publish Skills

Modify skills based on a description, then handle the git workflow to land the changes.

Use memory-mcp's `read` tool to load the `required-environment-variables` memory (scope: global)
if you haven't already this session, and use those identities for all git/gh operations throughout.

## Argument handling

`$ARGUMENTS` describes what to change. Examples:
- `develop: add a Phase 6 for documentation generation`
- `code-review: increase the large diff threshold to 800 lines`
- `create a new skill called "tdd" that runs red-green-refactor cycles`
- `briefing: add a section for checking open draft PRs`

If `$ARGUMENTS` is empty, check for uncommitted changes in the skills repo and
publish those (legacy publish-only mode).

## Prerequisites

This skill operates on the skills repo at `~/.claude/skills`. It assumes:
- The repo is initialized and has `origin` pointing to GitHub
- Branch protection on `main` requires PRs (no direct push)

## Phase 1: Understand and plan

1. Parse `$ARGUMENTS` to understand the desired change
2. Read the target skill's SKILL.md and any relevant references
3. If creating a new skill, identify the right directory and structure
4. Summarize what you'll change before making edits

## Phase 2: Make the edits

1. Edit the skill files to implement the requested changes
2. For existing skills: use the Edit tool for targeted changes
3. For new skills: create the directory structure and SKILL.md
4. For reference files: update or create as needed
5. Review your edits — read the modified files back to verify correctness

## Phase 3: Detect and analyze all changes

1. `git status` in `~/.claude/skills`
2. If the working tree is clean — report "Nothing to publish" and stop
3. `git diff` to review all changes (staged and unstaged)
4. If currently on a feature branch with unpushed commits, include those too

Review every changed file and understand what each edit does.

## Phase 4: Group changes into logical units

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

## Phase 5: For each logical group, branch → commit → push → PR

Process each group sequentially. For each one:

### 5a. Ensure on the right branch
- Start from `main` (pull latest first)
- Create a new branch named after the change (e.g. `skill/develop-add-phase`,
  `skill/edit-skill-rename`). Use `skill/` prefix.
- If already on a `skill/*` branch that matches this group: stay on it

### 5b. Commit
- Stage only the files belonging to this logical group
- Write a descriptive commit message summarizing the change
- Multiple commits within a group are fine if they represent distinct steps

### 5c. Push
- Push to origin with `-u`

### 5d. Open or update PR
- If no PR exists for this branch: create one via `gh pr create`
  - Title: concise description of the skill change
  - Body: summary of what changed and why, with the standard footer
- If a PR already exists: update the description if new commits were added,
  or just report the PR URL

### 5e. Return to main
- `git checkout main` before starting the next group

## Phase 6: Wait for merge

After all PRs are created, report the PR URLs and **wait for the user** to confirm
each PR is merged. Do not switch back to main or pull until the user says the PR(s)
are merged.

## Phase 7: Return to main

Once the user confirms the PR(s) are merged:
1. `git checkout main && git pull`
2. Confirm the working tree is clean and up to date

## Output format

After creating PRs:
```
## Skill changes published

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
