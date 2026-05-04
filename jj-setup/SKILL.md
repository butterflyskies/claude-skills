---
name: jj-setup
description: "Bootstrap jj configuration for a repo — installs revset aliases (stack(), bookmark_base()), snapshot settings, and auto-track config. Run once per repo. Requires jj >= 0.36."
---

# /jj-setup — Bootstrap jj Configuration

Install the required jj configuration for the stacking workflow. Run this once
per repo (or per user config if you want it globally). Idempotent — safe to
re-run.

## What it configures

### 1. Version check

Verify jj >= 0.36 (required for concurrent operation safety and auto-track):

```bash
jj --version
```

If below 0.36, stop and tell the user to upgrade. The concurrent operation
race condition fix (0.36) and `auto-track-created-bookmarks` (0.38) are
both required.

### 2. Revset aliases

```bash
jj config set --repo revset-aliases.'bookmark_base()' 'latest(::@ & remote_bookmarks())'
jj config set --repo revset-aliases.'stack()' 'bookmark_base()..@'
```

- `bookmark_base()` — most recent ancestor of `@` tracked on a remote
- `stack()` — all changes between `bookmark_base()` and `@`

These are used throughout `/develop`, `/land`, `/code-review`, and `/edit-skill`.

### 3. Snapshot settings

```bash
jj config set --repo snapshot.auto-update-stale true
```

Ensures workspaces staled by cross-workspace rebases auto-recover on the next
`jj` command. Required for multi-agent workflows where the coordinator may
rebase a stack that sub-agent workspaces descend from.

### 4. Bookmark auto-tracking

```bash
jj config set --repo remotes.origin.auto-track-created-bookmarks 'glob:pr/*'
```

Locally-created `pr/` bookmarks are automatically tracked with origin, so
`jj git push --bookmark pr/<name>` works without the deprecated `--allow-new`
flag.

### 5. Verification

After setting config, verify the aliases work:

```bash
jj log -r 'stack()' --no-graph -T 'change_id ++ "\n"' 2>&1 || true
jj log -r 'bookmark_base()' --no-graph -T 'change_id ++ "\n"' 2>&1 || true
```

If `bookmark_base()` resolves to nothing (no remote bookmarks in ancestry),
that's expected for a fresh repo — the skills fall back to `trunk()..@`.

Report what was configured and the jj version.

## Scope options

| `$ARGUMENTS` | Behavior |
|--------------|----------|
| *(empty)* | Configure the current repo (`--repo`) |
| `global` | Configure user-wide (`--user`) — applies to all repos |
| `check` | Report current config state without changing anything |
