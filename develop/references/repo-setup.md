# Repository Setup Conventions

Standard repo configuration applied to all new projects.

## GitHub branch protection (main)

Configure via `gh api` after repo creation:

1. **Require pull request before merging** — no direct pushes to main
2. **Require linear history** — enforces rebase or squash (no merge commits cluttering history)
3. **All merge types allowed** — merge, squash, and rebase all permitted (let the author choose)
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

## Git workflow

- **Feature branches required** — all work on `feature/<description>` or `fix/<description>`
- **Never commit directly to main**
- **Linear history** — rebase onto main before merging, or squash merge

## Pre-push quality gates

Enforced via git hooks (`.git/hooks/pre-push`) or CI:

### Rust
```bash
#!/usr/bin/env bash
set -euo pipefail
cargo fmt -- --check || { echo "Run cargo fmt first"; exit 1; }
cargo clippy -- -D warnings || { echo "Fix clippy warnings"; exit 1; }
cargo nextest run --workspace || { echo "Tests failing"; exit 1; }
```

### General
- Format check (language-specific formatter)
- Lint check (zero warnings)
- Test suite passes
- Build succeeds

These gates run automatically before any push to remote. Code that doesn't pass
format + lint + tests does not leave the local machine.

## CI (GitHub Actions)

Minimal CI that mirrors the local pre-push hooks:

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

The `cargo doc --no-deps` step in Phase 5b of the develop skill verifies docs build
cleanly before committing — the CI job here handles publishing to Pages.
