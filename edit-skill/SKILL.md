---
name: edit-skill
description: "Edit and publish skill changes. Takes a description of what to change as arguments, makes the edits, then handles the jj/PR workflow. Use when you need to modify an existing skill or create a new one."
---

# /edit-skill — Edit and Publish Skills

Modify skills based on a description, then handle the jj workflow to land the changes.

This skill is jj-native. Each logical group of skill changes becomes its own jj change
with its own `pr/<slug>` bookmark, landing as its own PR. See
[references/stacking-conventions.md](../references/stacking-conventions.md) for the
bookmark and PR mechanics.

Use memory-mcp's `read` tool to load the `required-environment-variables` memory (scope: global)
if you haven't already this session, and use those identities for all jj/git/gh operations throughout.

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
- The repo is a colocated jj repo (`jj git init --colocate` was run, or it was cloned
  with `jj git clone`)
- Branch protection on `main` requires PRs

## Phase 1: Understand and plan

1. Parse `$ARGUMENTS` to understand the desired change
2. Read the target skill's SKILL.md and any relevant references
3. If creating a new skill, identify the right directory and structure
4. Summarize what you'll change before making edits

Before making changes, check the working position:

```bash
jj log -r 'stack()'
```

If `stack()` is non-empty, ask whether the new edits build on the existing stack
or should be a fresh stack. If fresh, run `jj new bookmark_base()` to start from
the last published bookmark.

## Phase 2: Make the edits

1. Edit the skill files to implement the requested changes
2. For existing skills: use the Edit tool for targeted changes
3. For new skills: create the directory structure and SKILL.md
4. For reference files: update or create as needed
5. Review your edits — read the modified files back to verify correctness

jj snapshots the working copy when any `jj` command runs — no `git add` needed.
Run `jj util snapshot` to force a snapshot if needed. The current change
accumulates your edits.

## Phase 3: Detect and analyze all changes

1. `jj st` in `~/.claude/skills`
2. If the working tree is clean and `stack()` is empty — report "Nothing to publish"
   and stop
3. `jj diff -r 'stack()'` to review all changes across the current stack
4. If the stack has multiple layers already (from earlier edits this session), each
   one is its own group

Review every changed file and understand what each edit does.

## Phase 4: Group changes into logical units

Analyze the changes and decide how to split them:

- **One logical change** (e.g. a single skill edited, or tightly related edits across files):
  one change in the stack, one bookmark, one PR.
- **Multiple independent changes** (e.g. edits to unrelated skills, a new skill plus a
  separate fix to an existing one): separate changes in the stack (one per group),
  one bookmark per change, one PR per change.

Guiding principles:
- Each PR should be reviewable on its own — a coherent, self-contained change
- Edits to the same skill that serve the same purpose belong together
- A new skill is its own group unless it was created alongside tightly coupled edits to
  an existing skill
- When in doubt, split — smaller PRs are easier to review and merge

If the working copy currently mixes multiple groups, split them with `jj split`:

```bash
jj split  # interactive: select files for the first group
# repeat for subsequent groups; jj creates a chain of changes
```

If the groups are already separate (e.g., you edited skill A, ran `jj new`, edited
skill B), the stack is already in the right shape — just verify with `jj log -r 'stack()'`.

Summarize the grouping before proceeding (e.g. "I see two independent changes: X and Y.
I'll create separate PRs for each.").

## Phase 5: Describe each change

For each change in the stack, ensure it has a meaningful description. Run
`jj log -r 'stack()'` to see current descriptions; for any change with a
placeholder or empty description:

```bash
jj describe -r <change-id> -m "<descriptive message>"
```

Use the same conventions as repo commit messages — short subject line, optional
body explaining why.

## Phase 6: Land the stack

Invoke `/develop land` to handle bookmarks, push, and PR creation:

```
/develop land
```

This sets `pr/<slug>` bookmarks on each change in the stack, pushes them, opens
PRs with correct bases, and generates stack metadata in PR bodies. The slugs come
from the change descriptions — `/develop land` will ask you to confirm or override.

For skill PRs specifically, suggest slugs of the form `pr/skill-<name>-<intent>`
(e.g., `pr/skill-develop-add-jj-stacking`, `pr/skill-edit-skill-stack-mode`).

## Phase 7: Wait for merge

After all PRs are created, report the PR URLs and **wait for the user** to confirm
each PR is merged. Do not run further jj commands or fetch until the user says
the PR(s) are merged.

When the user reports a merge:

```bash
jj git fetch
jj abandon <change-id-of-merged-layer>  # optional cleanup
```

If the merged layer was the bottom of a stack, the next layer's `bookmark_base()`
now resolves to `main`. The stack continues from there.

## Output format

After invoking `/develop land`:
```
## Skill changes published

### Stack
- pr/skill-<slug-1>: <title>  →  PR #<num>
- pr/skill-<slug-2>: <title>  →  PR #<num>

Let me know when merged and I'll fetch and clean up.
```

After user confirms merge:
```
Fetched main. Stack updated. <N> layers landed; <M> remaining.
```
