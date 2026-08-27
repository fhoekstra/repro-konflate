# Issue: changed-only render drops producer for parent-injected `substituteFrom`

## Summary

When a parent Flux Kustomization uses `spec.patches` to inject `postBuild.substituteFrom` into child Kustomizations, flate's changed-only mode (`--path-orig` / `--base`) does not include the producer Kustomization that renders the referenced ConfigMap. The child Kustomization therefore reconciles before the ConfigMap exists. Because the injected `substituteFrom` is `optional: true`, missing variables are silently replaced with empty strings. If the rendered YAML uses those variables in mapping keys, the substituted document contains duplicate keys and fails to unmarshal.

The same failure occurs in **konflate**, because konflate always renders PRs with `orchestrator.RenderTrees`, which enables changed-only mode internally.

## Reproduction

> Use a fresh `--cache-dir` for each run. flate caches rendered sources across invocations, and a cached `cluster-settings` render can mask the failure on the failing branch.

Repository layout:

```text
flux/root/ks.yaml          # root Kustomization, patches substituteFrom into every child KS
clusters/kustomization.yaml
clusters/settings/ks.yaml  # Kustomization that produces cluster-settings ConfigMap
clusters/settings/cluster-settings.yaml
clusters/app/ks.yaml       # leaf Kustomization; substituteFrom is added by the parent patch
clusters/app/resources/consumer.yaml  # uses variables from cluster-settings
```

See this repo for the actual files. There are three branches:

* `main` — baseline
* `feature/repro-failing` — only `consumer.yaml` is changed; the leaf KS has no explicit `substituteFrom`
* `feature/repro-working` — same change plus an explicit `postBuild.substituteFrom` in the leaf KS

### Failing case

```bash
git checkout feature/repro-failing
rm -rf /tmp/flate-repro-cache
flate build ks --path flux/root --base main --cache-dir /tmp/flate-repro-cache -o yaml
```

**Expected:** the leaf `consumer` Kustomization renders successfully, substituting `READONLY_USER` and `GUEST_USER` from the `cluster-settings` ConfigMap.

**Actual:**

```text
✗  Kustomization  flux-system/consumer  substitute: unmarshal doc: yaml: construct errors: line 10: mapping key "/networks/ip" already defined at line 7; line 11: mapping key "/password" already defined at line 8; line 12: mapping key "/profile" already defined at line 9
```

The duplicate keys appear because `READONLY_USER` and `GUEST_USER` expanded to empty strings after the `cluster-settings` ConfigMap was not available.

### Working case

```bash
git checkout feature/repro-working
rm -rf /tmp/flate-repro-cache
flate build ks --path flux/root --base main --cache-dir /tmp/flate-repro-cache -o yaml
```

This succeeds and produces the correctly substituted `consumer` ConfigMap.

## Root cause

The leaf Kustomization manifest (`clusters/app/ks.yaml`) does not itself declare `postBuild.substituteFrom`. The reference is added at render time by the parent `cluster-apps` Kustomization patch. flate's changed-only filter builds the keep set before the parent patch is applied, so it never sees the hard `substituteFrom` edge from the leaf to `cluster-settings`. Consequently the `cluster-settings` Kustomization is not kept, and the ConfigMap is not materialized before the leaf reconciles.

## Workaround

Add the `postBuild.substituteFrom` reference directly to the leaf Kustomization manifest (even if the parent patch also adds it). This makes the dependency visible to the changed-only filter, which then pulls the producer into the render.

## Environment

* flate version: `0.6.0`
* konflate uses the same flate version and calls `orchestrator.RenderTrees`, so the failure reproduces there as well.
