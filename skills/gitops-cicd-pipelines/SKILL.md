---
name: gitops-cicd-pipelines
description: How to structure Git-driven deployment for multi-service apps — GitOps repo/folder layout, environment promotion with Argo CD + Kargo + Helm, and selective "deploy only what changed" pipelines for monorepos (Jenkins/GitHub Actions + Docker Compose + Traefik). Covers the change-detection traps that silently break these pipelines (shallow clone, shared-package edits that fire no path filter), plus the observability wiring to see the result (Prometheus receiver -> OTLP exporter via the OpenTelemetry Collector). Use when setting up or debugging a deploy pipeline for more than one service, when a push rebuilds everything and it shouldn't, when deciding monorepo vs polyrepo for delivery, when promoting a build dev -> staging -> prod, or when someone asks how to make deployments auditable and reproducible.
---

# GitOps & selective CI/CD pipelines

Two problems that look like one: **how a change reaches an environment** (GitOps promotion) and **which services a change should touch** (selective deploy). Solve them separately.

## Core rule

> Git is the single source of truth for *desired state*; the pipeline's only job is to make the change **detectable** and the promotion **reviewable**. If you can't answer "which commit produced what is running in prod right now" from Git alone, it isn't GitOps yet.

Four principles: Git as single source of truth · declarative systems · immutable deployments · centralized change audit.

## Part 1 — Promotion (Argo CD + Kargo)

**Split of duties:** Argo CD *reconciles* (watches Git, syncs cluster to match). Kargo *promotes* (watches Git/image/Helm repos, and commits the version bump that Argo CD then picks up). Neither replaces your CI — CI still builds and pushes the image.

**Kargo's four objects:**
| Object | Role |
|---|---|
| **Warehouse** | Watches an image registry, discovers new tags (`v1.2.0`, `v1.2.1`…), stores metadata |
| **Freight** | The concrete artifact version to promote (an image, a Helm chart) |
| **Stage** | An environment (dev/staging/prod); on new Freight it rewrites the manifest under `env/<stage>/` |
| **PromotionPolicy** | Whether a stage advances automatically or needs a human |

Flow: image `v1.2.0` pushed → Warehouse detects → Freight created → dev Stage updates `env/dev/` → Argo CD syncs dev → verification (tests/metrics) passes → Kargo updates staging Helm values in Git → Argo CD syncs staging → same again for prod behind a manual PromotionPolicy.

**⛔ Git branching per environment is an anti-pattern here.** A `dev`/`staging`/`prod` branch triple invites merge drift and makes "what's in prod" a diff question. Use **one branch, environment folders** (`env/dev/`, `env/staging/`, `env/prod/`) with per-env Helm values, and let promotion be a commit that changes an image tag.

**Layout that works** (polyrepo: app code separate from config):
```
argocd/          # AppProject + one Application manifest per environment
env/dev/         # Helm values + manifests for dev
env/staging/
env/prod/
kargo/           # Warehouse, Stage, PromotionPolicy
```

Local rehearsal stack: Minikube + Argo CD + Kargo — worth doing before touching a real cluster.

## Part 2 — Selective deploy (only what changed)

For a monorepo, the pipeline computes the changed service set and deploys only those.

Shape of the Detect-Changes stage:
1. Diff against the base commit → changed file list.
2. Map each path to a service via a prefix regex (`^apps/services/([a-z0-9-]+)/`, `^apps/gateways/…`, `^apps/core/…`).
3. Collect into a **Set**, drop non-deployables (`*-e2e`).
4. Gate the Deploy stage on that set being non-empty; loop over it, pulling and recreating only those Compose services.

### The traps that break this silently

- **⛔ Shallow clone.** `actions/checkout` defaults to `fetch-depth: 1`, so `git diff HEAD~1 HEAD` resolves against history that isn't there — you rebuild everything or nothing, nondeterministically. Set **`fetch-depth: 0`**, or explicitly fetch the merge base. Same class of bug in any runner that shallow-clones.
- **⛔ Shared packages fire no path filter.** Edit a library four services import: no service-directory filter matches, so nothing rebuilds and four services keep running against a library nobody rebuilt. Path filters are a *starting* heuristic — derive the affected set from the **dependency graph** once you have shared code.
- **`HEAD~1` is wrong on merge commits and on first-push branches.** Diff against the merge base of the target branch, not the previous commit.
- Untracked-but-required config outside the filtered paths (compose files, env templates) means "no service changed" yet the deploy is still needed — filter on those paths too.

### Operational gotchas (Docker/Jenkins/Traefik)
- `git config --global --add safe.directory <path>` — otherwise `fatal: detected dubious ownership` when the pipeline user ≠ tree owner.
- **`docker compose` (v2 plugin), not `docker-compose`** — the v1 shim rejects `-f` with `unknown shorthand flag`. Note where the plugin binary lives; distros differ (`/usr/lib/docker/cli-plugins/`).
- Editing `docker-compose.yml` and calling `restart` does **not** apply changes — you need `up -d` to recreate the container.
- Traefik serving its default self-signed cert = the ACME resolver never matched; check the router's TLS block and that DNS resolves publicly before debugging certs.
- Pin the container timezone or build timestamps and the CI UI disagree with your logs.
- Store the Git credential in the CI credential store (`withCredentials`), never inline in the Jenkinsfile.

## Part 3 — See the result (OpenTelemetry Collector)

Deploy pipelines are only trustworthy if you can watch what they shipped. To get existing **Prometheus** metrics into an OTLP backend without re-instrumenting the app:

`prometheus` receiver (scrapes your `/metrics`) → processors → `otlp` exporter → backend (SigNoz, or any OTLP-compatible store).

The subtlety is **histograms**: Prometheus exposes them as cumulative `_bucket`/`_sum`/`_count` series with `le` labels; OTLP wants a single histogram data point. The Collector's translation handles this, but bucket boundaries and the cumulative→delta choice must match what the backend expects — mismatches show up as missing or wildly wrong percentiles, not as errors. Verify end-to-end with generated test traffic before trusting a dashboard.

## Choosing the shape
| Situation | Go with |
|---|---|
| Several services, one team, shared libs | Monorepo + dependency-graph selective deploy |
| Independent release cadence, separate owners | Polyrepo + GitOps config repo |
| Manual approval required for prod | Argo CD + Kargo with manual PromotionPolicy |
| Small VPS, no Kubernetes | Jenkins/Actions + Docker Compose + Traefik (Part 2 only) |

Kubernetes is not a prerequisite for GitOps discipline — the folder-per-environment + promotion-as-a-commit model works on a single VPS too.

## Operational specs — intent separate from execution (added 2026-10-08)

A pipeline exiting 0 is not proof the operation succeeded (Hugo Teijiz, freeCodeCamp 2026-09-24 and 2026-09-29). Keep a vendor-neutral **spec** next to the pipeline:
- **Preconditions** (may we start?), **constraints** (must hold during/after, e.g. min available replicas, error-rate and latency ceilings), **evidence requirements** (which metrics/logs prove it), **recovery rules** (rollback/alert).
- After execution, collect the evidence and **evaluate against the spec** → pass/fail record = audit trail. Several executors (Argo, a script, an agent) can satisfy the same spec, so tool migration doesn't lose the rules.
- Matters most with AI agents executing ops: free in *method*, bounded by *intent*. A spec doesn't make a wrong threshold right — review it like code.

## Anti-patterns
- Branch-per-environment as the promotion mechanism → drift, and "what's in prod" becomes a diff.
- Rebuilding every service on every push because change detection was never made deterministic.
- Path filters over a shared-library monorepo → services shipping against un-rebuilt dependencies.
- Promotion by hand-editing an image tag in the cluster → the cluster and Git disagree and Argo CD reverts you.
- A pipeline that deploys but emits no metrics → you learn it broke from a user.
