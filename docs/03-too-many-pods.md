# 03 — Pending Pod: `Too many pods`

**Category:** Scheduling / node Pod capacity  
**Environment:** Single-node GKE lab  
**Status:** Observed during the lab

## The actual scheduler event

```text
Warning  FailedScheduling  default-scheduler
0/1 nodes are available: 1 Too many pods.
no new claims to deallocate, preemption: 0/1 nodes are available:
1 No preemption victims found for incoming pod.

Normal  NotTriggerScaleUp  cluster-autoscaler
Pod didn't trigger scale-up:
```

## What this tells us

- `0/1 nodes are available`: exactly one node was considered, and it could not host this Pod.
- `Too many pods`: the node's **Pod-count scheduling limit** was reached.
- `No preemption victims found`: the scheduler did not find an eligible lower-priority Pod whose removal would help.
- `NotTriggerScaleUp`: no cluster autoscaler scale-up was triggered for this Pod at that time. This message by itself does **not** prove why autoscaling did not occur.

This error is **not synonymous with CPU or memory exhaustion**. A node can have spare CPU and memory but still hit its maximum number of assigned Pods. GKE's system and monitoring workloads contribute to the total.

## Why there is a Pod-count limit

Every Pod needs management, networking, and node resources. Kubernetes exposes a per-node maximum in `status.capacity.pods` and `status.allocatable.pods`. The scheduler checks whether placing another Pod would violate the node's allocatable Pod capacity. The configured maximum depends on the cluster/network/node-pool configuration.

## Investigation

```bash
# See Pods in all namespaces, including GKE system Pods
kubectl get pods -A -o wide

# Show the node's advertised Pod capacity and allocatable count
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" capacity="}{.status.capacity.pods}{" allocatable="}{.status.allocatable.pods}{"\n"}{end}'

# Inspect node conditions, non-terminated Pods, and allocated resources
kubectl describe node <node-name>

# Re-read scheduling events on the affected Pod
kubectl describe pod <pending-pod-name>
```

`kubectl get pods -A` matters: counting only the `default` namespace would miss system workloads. Compare the Pod count **assigned to the specific node** with the advertised allocatable capacity. Do not equate the number of Running application Pods with total occupancy.

## Recovery choices

**Disposable single-node lab:** Remove temporary Pods or scale the application back to one replica *if appropriate*:

```bash
kubectl scale deployment/web --replicas=1
kubectl get pods -A -o wide
```

**Production:** Evaluate node-pool scale-out and per-node Pod density settings, then check whether cluster autoscaler can create compatible nodes within quota, IP-address, and policy constraints. Avoid changing Pod density or evicting production workloads without assessing networking and reliability consequences.

**After the change:** Verify new Pods become scheduled and Ready. If still Pending, inspect fresh events; another constraint may remain.

## Why we didn't assume the autoscaler was broken

Our event line said only `Pod didn't trigger scale-up:` with no explanatory text after the colon. We did not collect enough data to conclude whether autoscaling was disabled, limited, or blocked by a different constraint. Keep the conclusion specific to the evidence.

## Compare with CPU scheduling failures

| Event | Main constraint | First thing to compare |
| --- | --- | --- |
| `Too many pods` | Per-node Pod-count cap | Assigned Pod count vs. `allocatable.pods` |
| `Insufficient cpu` | CPU requests exceed per-node available allocatable CPU | Request of **one Pod** vs. capacity available on **one node** |
