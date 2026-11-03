# 05 — Deployment scaling, rolling updates, and Pod replacement

**Category:** Workload operations / reconciliation  
**Environment:** GKE, test Deployment `web`  
**Status:** Lab workflow and operational notes

## Goal

See how Kubernetes maintains a **desired state**, rather than treating a Pod as a permanent machine or process. A Deployment declares a desired number of replicas, and controllers continually compare desired state with actual state.

## Test A: scale replicas

```bash
kubectl scale deployment/web --replicas=3
kubectl get deployment web
kubectl get pods -o wide
```

**Expected controller flow:** The Deployment specifies the intended replica count, the ReplicaSet creates the necessary Pods, and the scheduler places them if suitable node capacity exists. **Three replicas do not imply three worker nodes**; multiple Pods can share one node.

If a Pod remains Pending, inspect the Pod's events (`kubectl describe pod ...`) to determine whether the blocker is CPU, memory, Pod count, taints, selectors, or something else.

Return to the original replica count for a small lab:

```bash
kubectl scale deployment/web --replicas=1
kubectl get pods
```

## Test B: delete a Deployment-managed Pod

```bash
kubectl get pods
kubectl delete pod <pod-name>
kubectl get pods -w
# Ctrl+C to exit the watch
kubectl get deployment web
```

**Why does a new Pod appear?** The ReplicaSet controller notices that the actual number of non-terminating Pods is below the requested count and creates a replacement. The scheduler then finds a node, and kubelet starts the container.

This is **self-healing at the Pod level**, not proof of zero downtime. A single replica can still create an interruption, and a replacement may fail for an unrelated reason.

## Test C: a rolling update

Any change to the Deployment's Pod template (for example, the image or CPU requests) causes a new ReplicaSet revision. With a RollingUpdate strategy, old and new ReplicaSets can coexist temporarily. The strategy's `maxSurge` and `maxUnavailable` influence how many Pods can be added or taken down during the transition.

```bash
kubectl rollout history deployment/web
kubectl get replicaset
kubectl rollout status deployment/web --timeout=60s
kubectl describe deployment web
```

Use these commands to understand **which revision** owns each Pod, **which revision** is stuck, and whether the Deployment still has healthy available replicas.

## What 'healthy' means

- **Scheduled**: assigned to a node.
- **Running**: at least one container is running or starting/restarting; not proof of application health.
- **Ready**: readiness conditions indicate the Pod is eligible for Service traffic.
- **Available**: a ready Pod has satisfied the Deployment's minimum availability duration, if configured.

Use readiness probes for meaningful application health; inspect Pod conditions and events when the status is ambiguous.

## Safe cleanup

Prefer restoring the desired replica count rather than manually deleting many Pods:

```bash
kubectl scale deployment/web --replicas=1
kubectl rollout status deployment/web --timeout=60s
kubectl get pods -o wide
```

**Operational lesson:** Change the **controller-owned desired state** to make durable changes. Deleting an individual managed Pod is temporary by design.
