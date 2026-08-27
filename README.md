# konflate / flate changed-only `substituteFrom` reproduction

Minimal repro for the issue described in [ISSUE.md](./ISSUE.md).

## Branches

* `main` — baseline
* `feature/repro-failing` — parent patch injects `substituteFrom`; leaf KS has no explicit reference
* `feature/repro-working` — leaf KS explicitly declares `postBuild.substituteFrom`

## Reproduce

```bash
# Failing case
git checkout feature/repro-failing
rm -rf /tmp/flate-repro-cache
flate build ks --path flux/root --base main --cache-dir /tmp/flate-repro-cache -o yaml

# Working case
git checkout feature/repro-working
rm -rf /tmp/flate-repro-cache
flate build ks --path flux/root --base main --cache-dir /tmp/flate-repro-cache -o yaml
```

The failing case exits non-zero with duplicate `/networks/ip`, `/password`, and `/profile` mapping keys because `READONLY_USER` and `GUEST_USER` expand to empty strings in changed-only mode.
