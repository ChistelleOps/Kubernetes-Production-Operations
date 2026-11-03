# 08 — Memory rises toward its limit with no restarts

**Category:** Observability / memory growth / incident prevention  
**Status:** Hypothetical production scenario — **not reproduced** in the GKE lab

> ⚠️ **Not verified in the lab.** This is a reasoned scenario. Treat commands and conclusions as a starting point and validate them in your environment.

## Scenario

A Pod is `Running`, with **0 restarts**. Its memory request is `256Mi` and limit is `1Gi`. Grafana shows memory usage around **900Mi**, increasing steadily over six hours, while request traffic remains stable.

**Operational question:** Do we restart the Pod, increase its limit, or investigate first?

## Why this needs attention

The service has not yet crashed, but it is nearing its limit. A further increase may cause an OOM termination. Stable traffic plus rising memory can indicate a memory leak, but other explanations include caching, queued work, changes in request size, a new release, or delayed garbage collection.

**Do not conclude there is a leak from a single memory graph.** Also check the metric definition: working set, RSS, heap allocation, and cache measure different things.

## Triage: establish impact and time to failure

```bash
kubectl get deployment -n <namespace> <deployment>
kubectl get pods -n <namespace> -o wide
kubectl describe pod -n <namespace> <pod-name>
kubectl logs -n <namespace> <pod-name> --tail=100
kubectl top pod -n <namespace> <pod-name> --containers
```

1. Is it **one replica or all replicas**? Is traffic served normally? Any increasing response latency or 5xx errors?
2. How quickly is memory rising? A stable 900Mi plateau is different from 900Mi on a steep upward trend.
3. Were there recent **code, library, configuration, or workload** changes? Is a specific request or job correlated with growth?
4. Compare memory metrics (working set, RSS, heap where available), garbage collection, open connections, queues, and cache size.
5. Check current node memory pressure and headroom; do not assume raising the limit is safe.

Example *diagnostic* PromQL (adapt labels to your telemetry). Working set is the figure the kernel compares against the limit:

```promql
# Working-set memory as a fraction of the container's limit
container_memory_working_set_bytes{namespace="demo",container="app"}
/
kube_pod_container_resource_limits{namespace="demo",container="app",resource="memory"}

# Growth rate over the last 6 hours (bytes/second)
deriv(container_memory_working_set_bytes{namespace="demo",container="app"}[6h])
```

A positive `deriv` that persists through traffic troughs is stronger evidence than the raw level. It is still not proof of a leak.

## Decide how to mitigate

- **No immediate disruption and low short-term risk:** continue observation, collect diagnostic evidence, and have app owners investigate. Set a near-term alert.
- **High risk of OOM or user impact:** prepare a controlled mitigation based on known-safe behavior, such as reverting a memory-regressing release or safely restarting one replica at a time. Consider a temporary limit change **only with capacity validation**.
- **Suspected leak:** capture profiles/heap diagnostics following application-specific safe procedures. Restarting can erase valuable evidence.

A restart might lower memory temporarily without fixing the cause. A higher limit may only delay the crash.

## Verification and closure

After mitigation, compare memory growth over a meaningful interval, not just a single `kubectl top` snapshot. Confirm readiness, restart count, availability, and user-facing latency. Document the cause or explicitly mark it **unconfirmed** when evidence is insufficient.

## Related

- [07 — OOMKilled](07-oomkilled-memory.md): what happens if the limit is reached.

## Prevention

Alert on sustained memory growth and remaining headroom; retain memory and deployment history; include realistic memory behavior in load and soak tests; and give application teams safe profiling procedures.
