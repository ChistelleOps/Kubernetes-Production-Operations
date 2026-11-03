# 06 — CPU throttling: container tries to use more than its CPU limit

**Category:** Compute / Linux cgroups / performance  
**Status:** Conceptual production scenario — **not reproduced** in the GKE lab

> ⚠️ **Not verified in the lab.** This is a reasoned scenario. Treat commands and conclusions as a starting point and validate them in your environment.

## Scenario and symptom

A container is configured with a CPU request of `500m` (half a core) and a CPU limit of `1000m` (one core). Its application tries to use approximately **1.5 cores** while processing requests. The Pod stays `Running`, but latency increases.

```yaml
resources:
  requests:
    cpu: 500m
  limits:
    cpu: 1000m
```

The question to investigate is: **why does the application slow down rather than crash?**

## How Kubernetes and Linux work together

- The **scheduler** uses the CPU *request* to decide whether a Pod fits on a node. It does not schedule against the CPU *limit*.
- The kubelet/container runtime configures Linux **cgroups**. CPU limits are normally enforced through a CFS bandwidth quota (or equivalent cgroup CPU control), which restricts CPU time over repeated periods.
- When the container exhausts its allowed CPU time in a period, it is **throttled** until it can run again. Its process generally remains alive.
- **CPU throttling is not OOMKilled.** OOM killing concerns memory, not exceeding CPU quota.

A request of `500m` is a scheduling guarantee under CPU contention, **not** a claim that the container always uses 0.5 CPU. A limit of `1000m` is a maximum average CPU budget, not a promise of one dedicated physical core.

## Investigation: commands and reasons

```bash
kubectl get pod -n <namespace> <pod-name> -o wide
kubectl describe pod -n <namespace> <pod-name>
kubectl top pod -n <namespace> <pod-name> --containers
kubectl describe node <node-name>
```

1. Check the Pod's **requests and limits** in `describe`.
2. Use `top` for recent CPU usage, noting that it does **not** report throttling directly and depends on a metrics API.
3. Correlate Grafana latency, request rate, CPU utilization, and throttling over the same time window.
4. In Prometheus, when available, inspect `container_cpu_cfs_throttled_periods_total`, `container_cpu_cfs_periods_total`, and `container_cpu_cfs_throttled_seconds_total`. Filter by workload/container and exclude sandbox/empty containers as needed. Metric availability varies with the runtime and monitoring configuration.
5. Check whether throttling began after a deployment or resource-setting change, and whether the app is CPU-intensive or blocked on something else.

Example *diagnostic* PromQL (adapt labels to your telemetry):

```promql
rate(container_cpu_cfs_throttled_periods_total{namespace="demo",container="app"}[5m])
/
rate(container_cpu_cfs_periods_total{namespace="demo",container="app"}[5m])
```

This estimates the fraction of CFS periods that experienced throttling; it **does not** directly represent percentage CPU lost or prove user-visible impact on its own.

## Resolution choices

- If latency is caused by sustained CPU throttling and the node has room, evaluate a **higher CPU limit** after load testing.
- If the application needs dedicated, predictable CPU capacity, reconsider requests, node sizing, and CPU management policy.
- If the application wastes CPU, profile and optimize it before buying more compute.
- For controlled environments, removing a CPU limit may be appropriate under a resource governance policy, but requires capacity and noisy-neighbor considerations.

**Verify:** deployment Ready, latency normalized, error rates stable, and throttling reduced during comparable load. Record the resource change in version control.

## Do not confuse these failures

| Symptom | Likely area |
| --- | --- |
| Pod `Pending`, event `Insufficient cpu` | Scheduler cannot fit **request** |
| Pod `Running`, slow under load, high throttling | CPU **limit** or CPU contention |
| Container terminated, `OOMKilled` | Memory exhaustion / memory limit |

## Related

- [04 — Insufficient cpu](04-insufficient-cpu.md): requests and the scheduler.
- [07 — OOMKilled](07-oomkilled-memory.md): memory limits kill rather than throttle.

## Prevention

Establish requests from observed demand, test limits under load, dashboard CPU throttling and latency, and review changes through Git. Avoid treating a request and a limit as interchangeable.
