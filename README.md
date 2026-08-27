# konflate / flate changed-only `substituteFrom` reproduction

Minimal repro for the issue described in [ISSUE.md](./ISSUE.md).

## Branches

* `main` — baseline
* `feature/repro-failing` — parent patch injects `substituteFrom`; leaf KS has no explicit reference

## Reproduce locally

```bash
git checkout feature/repro-failing
rm -rf /tmp/flate-repro-cache
flate build ks --path flux/root --base main --cache-dir /tmp/flate-repro-cache -o yaml
```

This exits non-zero with duplicate `/networks/ip`, `/password`, and `/profile` mapping keys because `READONLY_USER` and `GUEST_USER` expand to empty strings in changed-only mode.

## konflate note

This command renders only one side. **konflate** uses `orchestrator.RenderTrees`, which renders both the PR head and the merge-base in changed-only mode. The same failure reproduces there because the dependency edge is injected by a parent patch and is not visible to flate's changed-only keep-set builder.
