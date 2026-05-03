# Repository Setup Conventions

Standard repo configuration applied to all new projects. Repos are colocated jj+git
(`jj git init --colocate` or `jj git clone`) so the on-disk format is git but local
operations use jj.

## GitHub branch protection (main)

Configure via `gh api` after repo creation:

1. **Require pull request before merging** — no direct pushes to main
2. **Require linear history** — enforces squash merge (no merge commits cluttering
   history). This is the natural fit for stacked PRs.
3. **Squash merge only** — disable "merge commit" and "rebase and merge" in repo
   settings. Squash merge keeps the main-branch history clean (one commit per PR)
   and jj's content-aware rebase handles the local/remote SHA divergence naturally
   (see stacking-conventions.md "Squash-merge: what happens to the local change").
4. **No force-push to main** — ever

```bash
gh api repos/{owner}/{repo}/rulesets --method POST --input - <<'EOF'
{
  "name": "main-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main"],
      "exclude": []
    }
  },
  "rules": [
    {"type": "pull_request", "parameters": {"required_approving_review_count": 0, "dismiss_stale_reviews_on_push": false, "require_last_push_approval": false}},
    {"type": "required_linear_history"},
    {"type": "non_fast_forward"}
  ]
}
EOF
```

Note: `required_approving_review_count: 0` means a PR is required but no human approval
is needed — the AI can self-merge after CI passes. Adjust per repo if human review is wanted.

## Workflow conventions

- **Stack-based development** — work happens in a chain of jj changes built on top of
  the most recent published bookmark. Each layer in the stack lands as its own PR.
  See [../../references/stacking-conventions.md](../../references/stacking-conventions.md).
- **Never commit directly to main** — `main` is the published trunk. All work is on
  `pr/<slug>` bookmarks pushed via `jj git push`.
- **Squash merge** — all PRs land via squash merge. jj's content-aware rebase
  handles the local/remote SHA divergence naturally.

## Pre-push quality gates

Enforced via a `jj git push` wrapper or CI. There is no native pre-push hook in jj,
but you can add a shell wrapper:

```bash
# ~/.local/bin/jj-push (or similar)
#!/usr/bin/env bash
set -euo pipefail
cd "$(jj workspace root)"
cargo fmt -- --check || { echo "Run cargo fmt first"; exit 1; }
cargo clippy -- -D warnings || { echo "Fix clippy warnings"; exit 1; }
cargo nextest run --workspace || { echo "Tests failing"; exit 1; }
exec jj git push "$@"
```

Note: `.git/hooks/pre-push` does **not** fire on `jj git push` — jj bypasses git
hooks entirely (it writes directly to the git object store). The shell wrapper
above is the only reliable local pre-push gate.

These gates enforce: format check, lint check (zero warnings), test suite passes,
build succeeds. Code that doesn't pass format + lint + tests does not leave the
local machine.

## CI (GitHub Actions)

Minimal CI that mirrors the local pre-push hooks. CI sees the colocated git repo —
it doesn't need jj installed, since pushed changes are normal git commits with normal
branch refs (the `pr/<slug>` bookmarks).

```yaml
name: CI
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Format
        run: cargo fmt -- --check
      - name: Clippy
        run: cargo clippy -- -D warnings
      - name: Test
        run: cargo nextest run --workspace
```

CI is the safety net — if local hooks were bypassed, CI catches it.

For stacked PRs, CI runs on each PR independently. Each layer's PR diff is just that
layer's commit, so CI verifies each layer compiles and passes tests on its own
foundation. This is what makes the stack reviewable: layer 2's CI failures aren't
hidden behind layer 1's content.

## Docs (GitHub Pages via Actions) — Rust

For Rust projects, publish `cargo doc` output to GitHub Pages on every push to main:

```yaml
name: Docs
on:
  push:
    branches: [main]
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: true
jobs:
  docs:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - name: Build docs
        run: cargo doc --no-deps --document-private-items
      - name: Add redirect
        run: echo '<meta http-equiv="refresh" content="0;url=<crate_name>/index.html">' > target/doc/index.html
      - uses: actions/upload-pages-artifact@v3
        with:
          path: target/doc
      - id: deployment
        uses: actions/deploy-pages@v4
```

Replace `<crate_name>` with the actual crate name (hyphens become underscores).
Enable GitHub Pages in repo settings → Pages → Source: GitHub Actions.

The `cargo doc --no-deps` step in `/develop` Phase 5b verifies docs build cleanly
before landing — the CI job here handles publishing to Pages.

## Initial repo setup

For a new project:

```bash
gh repo create butterflyskies/<name> --private
git clone <url> <name>
cd <name>
jj git init --colocate
# now you have a colocated jj+git repo
echo "# <name>" > README.md
jj describe -m "Initial commit"
jj bookmark set main -r @
jj bookmark track main@origin
jj git push --bookmark main
```

After the first push, configure branch protection (above) before any feature work.
