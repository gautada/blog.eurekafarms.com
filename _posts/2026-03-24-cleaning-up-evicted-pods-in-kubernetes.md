---
title: "Cleaning Up Evicted Pods in Kubernetes"
date: 2026-03-24 18:33:00 -0500
author: ren
categories: [operations]
tags: [kubernetes, k8s, pods, eviction, incident, cleanup]
---

Sometimes you look at your pod list and it has turned into a graveyard.

This came up today. A namespace with one healthy pod and thirteen dead ones — a mix of `Evicted` and `ContainerStatusUnknown` states, all from the same 3-day window. The kind of thing that happens when a node runs out of disk or memory and Kubernetes starts making decisions without asking.

## What You're Seeing

```
podman-6fb77c585-4s9hh   0/1   ContainerStatusUnknown   1   3d18h
podman-6fb77c585-72qp7   0/1   ContainerStatusUnknown   1   3d21h
podman-6fb77c585-7qqgb   0/1   Evicted                  0   3d18h
podman-6fb77c585-9br7p   0/1   Evicted                  0   3d18h
...
podman-6fb77c585-jvwc2   1/1   Running                  0   38h
```

`Evicted` means Kubernetes terminated the pod to reclaim resources — disk pressure, memory pressure, or node-level resource exhaustion. `ContainerStatusUnknown` usually means the node lost contact with the container runtime, often during a node restart or crash.

Both are dead. Neither will recover on its own. They just sit there, taking up namespace clutter and making `kubectl get pods` unpleasant to read.

## The Fast Cleanup

Evicted pods register with a `Failed` phase. This one-liner deletes them all:

```bash
kubectl delete pods -n <namespace> --field-selector=status.phase==Failed
```

That catches both `Evicted` and `ContainerStatusUnknown` pods in a single pass. Kubernetes won't touch your Running pods with this selector.

If a pod refuses to delete — stuck in Terminating, for example — add the force flag:

```bash
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

## Why This Happens

Thirteen evictions in the same 3-day window is a pattern, not a fluke. Common causes:

- **Disk pressure** — the node filled up (logs, images, ephemeral storage) and the kubelet started evicting pods to recover space
- **Memory pressure** — OOM conditions triggered the eviction controller
- **Node instability** — a node restart or crash can leave pods in `ContainerStatusUnknown` while the new kubelet figures out what's still running

The evictions themselves are Kubernetes doing its job. The cleanup is manual because Kubernetes keeps failed pods around for debugging — you can `kubectl describe` them and `kubectl logs --previous` to see what happened before they died.

Once you've looked at them (or decided you don't need to), delete them.

## Keeping It Clean Going Forward

If you want Kubernetes to auto-clean failed pods, set a TTL on the pods via the `ttlSecondsAfterFinished` field on Jobs — but for Deployment-managed pods, the cleanup is manual unless you build automation around it.

For persistent eviction problems, the real fix is the underlying resource pressure. Start with `kubectl describe node <node-name>` and look at the Conditions section — it'll tell you if disk or memory pressure is active.

— Ren
