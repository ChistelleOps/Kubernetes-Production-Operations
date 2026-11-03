# 07 — `OOMKilled`: container terminates after memory exhaustion

**Category:** Memory / container runtime / reliability  
**Status:** Conceptual production scenario — **not reproduced** in the GKE lab

> ⚠️ **Not verified in the lab.** This is a reasoned scenario. Treat commands and conclusions as a starting point and validate them in your environment.

## Scenario and symptom

An application has `requests.memory: 256Mi` and `limits.memory: 512Mi`. Its container restarts repeatedly. In a Pod description:

```text
Last State:  Terminated
Reason:      OOMKilled
Exit Code:   137
```

```yaml
resources:
  requests:
    memory: 256Mi
  limits:
    memory: 512Mi
```

The fact that the **request is lower than the limit is normal**: the request is used for placement, while the memory limit is enforced at runtime. A container can use more than its request. If a memory allocation cannot be satisfied within its cgroup limit, the Linux kernel can terminate a process in that cgroup.

`137` typically means the process was killed with SIGKILL (`128 + 9`), but **exit code 137 alone does not prove OOM**. Check the termination **reason** and events.

## Investigation workflow

### 1. Confirm restart reason and affected container

```bash
kubectl get pods -n <namespace>
kubectl describe pod -n <namespace> <pod-name>
kubectl get pod -n <namespace> <pod-name> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}'
```

Check restart count, limit, timestamps, last terminated state, and whether more than one container is affected.

### 2. Read prior logs safely

```bash
kubectl logs -n <namespace> <pod-name> -c <container-name> --previous --tail=100
```

`--previous` matters when the container has restarted. Logs may not show the actual OOM because the kernel terminates the process abruptly. Avoid printing credentials or user data into an incident record.

### 3. Review recent and historical memory metrics

```bash
kubectl top pod -n <namespace> <pod-name> --containers
kubectl top nodes
kubectl describe node <node-name>
```

`kubectl top` displays recent usage, not historical peaks. The restarted container may now look healthy. Use monitoring to inspect **memory working set**, restarts, limit changes, and traffic during the period just before termination. Review node `MemoryPressure` separately; a container-limit OOM can happen even without node-wide memory pressure.

### 4. Assess root cause

Investigate whether the memory limit is too low for normal workload, whether memory usage follows traffic or batch operations, whether a recent release added memory consumption, and whether memory keeps growing without stabilizing (possible leak). A rising graph is a clue, **not proof** of a leak.

## Safe recovery

- For an urgent incident, first assess impact, replica health, and recent changes. A known-good rollback may restore service if the release caused the growth.
- If historical normal usage genuinely exceeds the configured limit, adjust request/limit through the deployment source **after** checking node capacity and impact.
- If a leak is suspected, coordinate with developers on profiling and a fix. A controlled restart is only a temporary mitigation.
- Watch rollout progress, restart counts, errors, and memory trends after intervention.

```bash
kubectl rollout status deployment/<deployment> -n <namespace>
kubectl get pods -n <namespace> -w
```

## Related

- [08 — Memory rising toward its limit](08-growing-memory-possible-leak.md): the same problem before the container is killed.
- [06 — CPU throttling](06-cpu-throttling.md): exceeding a CPU limit throttles; exceeding a memory limit kills.

## Prevention

Instrument memory and restarts; set alerts on sustained growth relative to the limit; load-test representative traffic; and verify changes with post-release dashboards. Avoid blindly increasing memory without identifying why it is consumed.
