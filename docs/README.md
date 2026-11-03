# Kubernetes Production Operations: troubleshooting notes

Ten incident write-ups from a GKE lab. Find the symptom you are seeing, then open the matching doc.

**Verified** means we reproduced it in the GKE lab. **Scenario** means it is a reasoned production scenario that was not reproduced.

| Symptom | Doc | Status |
| --- | --- | --- |
| `gke-gcloud-auth-plugin` not found, `kubectl` can't authenticate (macOS) | [01 — GKE auth plugin](01-gke-issue.md) | Verified |
| `ErrImagePull` / `ImagePullBackOff` after an update | [02 — Invalid image tag](02-image-pull-backoff.md) | Verified |
| Pod `Pending`, event `Too many pods` | [03 — Too many pods](03-too-many-pods.md) | Verified |
| Pod `Pending`, event `Insufficient cpu` | [04 — Insufficient cpu](04-insufficient-cpu.md) | Verified |
| Scaling, self-healing and rolling-update basics | [05 — Self-healing](05-deployment-self-healing.md) | Verified |
| Pod `Running` but slow, latency up under load | [06 — CPU throttling](06-cpu-throttling.md) | Scenario |
| Container restarting, `OOMKilled`, exit code 137 | [07 — OOMKilled](07-oomkilled-memory.md) | Scenario |
| Memory climbing toward the limit, no restarts yet | [08 — Growing memory](08-growing-memory-possible-leak.md) | Scenario |
| `CrashLoopBackOff` with a database auth error in logs | [09 — Postgres auth](09-crashloopbackoff-postgres-auth.md) | Scenario |
| Secret updated but Pods still use the old credential | [10 — Secret rotation](10-secret-rotation-env-vars.md) | Scenario |

## Quick triage by Pod state

| Pod state | Look at |
| --- | --- |
| `Pending` | `kubectl describe pod`, Events: [03](03-too-many-pods.md), [04](04-insufficient-cpu.md) |
| `ImagePullBackOff` | Events for the real cause: [02](02-image-pull-backoff.md) |
| `CrashLoopBackOff` | `kubectl logs --previous`: [09](09-crashloopbackoff-postgres-auth.md), [07](07-oomkilled-memory.md) |
| `Running` but unhealthy | Metrics: [06](06-cpu-throttling.md), [08](08-growing-memory-possible-leak.md) |

## Common distinctions

- Requests are for scheduling. Limits are enforced at runtime ([04](04-insufficient-cpu.md), [06](06-cpu-throttling.md)).
- A CPU limit throttles. A memory limit kills ([06](06-cpu-throttling.md), [07](07-oomkilled-memory.md)).
- `rollout undo` reverts the Pod template only. It does not revert Secrets or database changes ([09](09-crashloopbackoff-postgres-auth.md), [10](10-secret-rotation-env-vars.md)).

**Security:** never paste Secret values, kubeconfigs or tokens into tickets, logs or Git.
