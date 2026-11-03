# 04 — Pending Pod: `Insufficient cpu` after a resource change

**Category:** Scheduling / resource requests  
**Environment:** GKE Standard, one `e2-medium` worker (2 vCPUs)  
**Status:** Deliberately reproduced and diagnosed

## The exact change

```bash
kubectl set resources deployment/web --requests=cpu=8
# deployment.apps/web resource requirements updated
```

This command changes the Deployment Pod template so **each newly created Pod requests 8 CPU cores**. `cpu=8` means 8 cores, **not 8 percent**. CPU also supports millicores: `100m` = 0.1 core; `500m` = 0.5 core; `1000m` = 1 core.

## What we observed

Two Pods belonging to the old ReplicaSet continued running:

```text
web-7887448d46-qpq8p   1/1   Running
web-7887448d46-s8fd8   1/1   Running
```

The new ReplicaSet created a Pod that stayed Pending:

```text
web-54849b6bbd-x24vh   0/1   Pending
```

The Pod description showed:

```text
Node:          <none>
Controlled By: ReplicaSet/web-54849b6bbd
Requests:
  cpu:        8
PodScheduled: False
Warning FailedScheduling: 0/1 nodes are available: 1 Insufficient cpu.
Normal  NotTriggerScaleUp: Pod didn't trigger scale-up:
```

## Why it failed

The scheduler chooses **one node** for each Pod. It uses resource **requests**, node allocatable capacity, and existing reservations to determine fit. Our only node had **2 vCPUs total** (and even less allocatable after system reservations). An 8-CPU request cannot fit there regardless of current CPU usage.

Even if several 2-vCPU nodes were present, they cannot combine into one 8-CPU machine for a single Pod. Scale-out with identical undersized nodes does not solve this specific request; a larger compatible node would be needed, or the request should be corrected.

## Why existing Pods kept running

Changing requests changed the **Pod template**, so the Deployment made a new ReplicaSet and started a **rolling update**. The old ReplicaSet's Pods were still healthy. The new Pod could not schedule, so the rollout stalled while old Pods continued running, subject to the Deployment's update strategy.

## Investigation

```bash
kubectl get deployment web
kubectl get replicaset
kubectl get pods -o wide
kubectl describe pod web-54849b6bbd-x24vh
kubectl describe node <node-name>
kubectl rollout history deployment/web
```

In `describe node`, distinguish:

- **Capacity:** gross node resources.
- **Allocatable:** resources available to Pods after node reservations.
- **Allocated resources:** total scheduled requests and limits, **not live CPU usage**.

`kubectl top nodes` shows observed usage (if metrics are installed) but is **not** the main evidence for a scheduler `Insufficient cpu` decision.

## Recovery used for an intentionally incorrect change

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web --timeout=60s
kubectl get pods
```

Rolling back restores the prior Pod template, including its previous resource settings. Verify Ready/Available replicas and check that the unfit new revision is no longer blocking the rollout.

For a real 8-core workload, choose a node type with at least 8 **allocatable** cores available for that Pod (often requiring more than 8 gross vCPUs once reservations/other Pods are considered). Update the canonical deployment configuration in Git after identifying the correct resource requirement.

## Related: limits are different from requests

```yaml
resources:
  requests:
    cpu: 500m
  limits:
    cpu: 1000m
```

Here the scheduler reserves **0.5 core** for fit; the running container is limited to **1 core** of CPU time. Attempts to use more than the CPU limit lead to **throttling**, not `OOMKilled`. Memory limits work differently: memory exhaustion can terminate a process/container.

## Related

- [06 — CPU throttling](06-cpu-throttling.md): what happens when a *running* container hits its CPU limit.
- [03 — Too many pods](03-too-many-pods.md): a different scheduler constraint.
- [02 — Invalid image tag](02-image-pull-backoff.md): another stalled rollout.

## Key lesson

`Pending` + `PodScheduled: False` + `Insufficient cpu` = **scheduler cannot place the Pod**. Investigate resource requests and per-node allocatable capacity before assuming the application itself is faulty.
