---
title: "Owning the Telemetry Stack at eureka!FARMS"
date: 2026-03-25 08:55:00 -0400
author: nyx
categories: [observability]
tags: [opentelemetry, tempo, loki, supply-chain, kubernetes]
---

We have two ways to stand up observability backends:

1. `helm install grafana/loki` (or tempo, or kube-prometheus-stack) and accept the upstream defaults.
2. Pull the source, build our own images, and manage the manifests like any other first-party service.

At eureka!FARMS, Path 2 is the default. It is slower on day one but pays off when the cluster is under load, an SBOM audit shows up, or we need to inject one more sidecar to satisfy platform policy.

This post explains why we keep rolling our own Tempo/Loki builds, where it hurts, and the guardrails that keep the pain tolerable.

## Why we own the builds

| Win | What it gives us |
| --- | --- |
| **Supply chain control** | We compile from tagged upstream source on our own base images. Every artifact is scanned, signed, and versioned in the same pipeline as our app containers. |
| **Policy compliance** | SecurityContext, PodSecurity, and network policies live beside the code. There is no mystery YAML hiding inside a chart template. |
| **Customization headroom** | Want to trim receivers, add feature flags, or wire OTLP directly into the service mesh? We change YAML, not Helm templating. |
| **Predictable operations** | Release cadence, topology, and config diffs are entirely ours. No surprises when a chart rename lands in `helm repo update`. |

## The real downsides (and how we blunt them)

| Friction | Mitigation |
| --- | --- |
| **Upstream drift** | A GitHub Action tracks Tempo and Loki releases. When a new tag lands, it opens an issue with the changelog and CVE deltas. We decide when to pick it up, but we never miss the notice. |
| **Re-creating chart logic** | We regularly `helm template` the official chart into a scratch branch, study the rendered topology, and "vendor" the bits we like. That keeps our manifests idiomatic without inheriting the template engine. |
| **YAML bloat** | Kustomize bases capture shared pieces (service accounts, PodSecurity, resource defaults). Overlays just patch storage classes, object-store creds, or replica counts. |
| **Testing burden** | Every telemetry change spins a KinD cluster in CI, applies the manifests, and fires `otel-cli` probes for traces, logs, and metrics. If any probe fails, the PR stops there. |
| **Human time** | All observability builds go through the same container pipeline as everything else. Dependency bumps, SBOM exports, and signing are centralized instead of bespoke scripts. |

## When Helm still wins

We do not ban charts. We use them to:

- Prototype a brand-new stack in a disposable namespace.
- Compare upstream defaults against our hardened configs.
- Stand up short-lived tooling (e.g., demo dashboards) where supply-chain constraints are lower.

Once a component becomes part of the production control plane, we migrate it into the build-from-source lane.

## Checklist before wiring Tempo/Loki to the collector

1. **Storage first.** Tempo wants object storage for blocks; Loki wants object storage for chunks plus a boltdb-shipper (or Dynamo/Bigtable) index. PVCs work only as a temporary stopgap.
2. **Service topology.** Decide between single-binary and microservice modes. We prefer microservice for Loki (ingester/distributor/query-frontend) and single-binary with compactor enabled for Tempo until scale forces a split.
3. **Baseline hardening.** Non-root UID, read-only rootfs, PodSecurity level `restricted`, network policies that only allow collector traffic in and object storage out.
4. **Observability of observability.** Enable Tempo and Loki's own metrics/logs and scrape them so we catch ingestion backpressure before developers do.
5. **Collector wiring.** Once the services are up, add OTLP and Loki exporters in the collector config and roll the deployment to pick up the new endpoints.

## Takeaway

Running upstream Helm charts is faster in the short term, but eureka!FARMS optimizes for long-lived, audit-friendly infrastructure. Owning the build gives us deterministic artifacts, consistent policies, and the freedom to make weird-but-necessary changes without negotiating with template logic.

The trade-off is engineering time. We pay that cost once, amortize it through automation, and sleep better when an incident hits.
