# Migration Checklist: Replacing Dependency X with Y

Use this checklist when replacing a dependency, rewriting a module, or swapping out an
implementation backend. The #1 source of multi-round reviews on migration PRs is implicit
behaviors in the old code that the new code silently drops.

## Before writing any replacement code

### 1. Behavioral inventory

For the old dependency/module, enumerate every behavior it provides. Check each category:

- [ ] **Error handling** — What errors does X produce? How does it handle invalid input?
      Does it panic, return Result, or silently succeed? Does Y handle the same cases?
- [ ] **Resource limits** — Does X have internal batch sizes, connection pools, queue depths,
      memory caps, or timeout defaults? These are often undocumented. Read the source.
- [ ] **Caching** — Does X cache results, models, connections, or computed values? Where?
      (Disk path, env var, in-memory?) Does the cache location change with Y?
- [ ] **Configuration** — What env vars, config files, or CLI flags does X read? Does Y use
      different ones? Will users' existing configuration silently stop working?
- [ ] **Internal chunking/batching** — Does X process inputs in chunks (e.g., batch size 256
      for embeddings, 1000-row INSERT batches)? If Y doesn't chunk, large inputs may OOM.
- [ ] **Thread safety** — Is X thread-safe? Is Y? If X was `Send + Sync` and Y isn't, callers
      sharing it across threads will break.
- [ ] **Panic safety** — Does X catch panics internally? If Y can panic (e.g., in unsafe code
      or via `unwrap`), a long-running server with `Mutex`-wrapped state will poison on panic
      and reject all future requests. Add `catch_unwind` if needed.
- [ ] **Implicit normalization** — Does X normalize inputs (trim whitespace, lowercase, pad,
      truncate)? If Y doesn't, outputs may differ subtly.

### 2. Write tests for each behavior

For each behavior identified above, write a test that:
- Exercises the behavior through the **public API** (not by reimplementing internals)
- Would **fail** if the new code omits that behavior
- Documents what it's testing in the test name: `test_<behavior>_preserved_after_migration`

### 3. Check configuration continuity

- [ ] List all env vars the old code reads → verify the new code reads the same ones (or
      documents the change in migration notes)
- [ ] Check Dockerfile / CI / deployment configs that reference old paths, model names, or
      cache directories
- [ ] Verify any user-facing CLI flags still work or are explicitly removed with a clear error

## During implementation

### 4. Implement with tests green

- Write the replacement, running the behavioral tests from step 2 as you go
- When a test fails, that's a behavior gap — fix it before moving on
- Don't delete the old code until all behavioral tests pass with the new code

### 5. Check for dropped safety nets

Ask explicitly:
- "What happens if the input is 10x larger than any test case?" (batch size, memory)
- "What happens if the operation panics?" (mutex poisoning, partial state)
- "What happens if the config env var isn't set?" (different defaults)

## After implementation

### 6. Integration test the full pipeline

- Don't just unit-test the new component in isolation
- Test the full flow that uses it: input → processing → output
- Compare outputs against the old implementation for representative inputs

### 7. Update deployment artifacts

- [ ] Dockerfile (cache paths, env vars, model downloads)
- [ ] CI config (new dependencies, build flags, test commands)
- [ ] Deployment manifests (resource limits, env vars, volume mounts)
- [ ] Documentation (README, ADRs, inline comments referencing old dependency)
