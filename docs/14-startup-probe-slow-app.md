# 14 — Slow application startup causes premature liveness restarts

**Type:** Hypothetical production scenario discussed for learning; **not tested on GKE**.  
**Area:** Startup, liveness, readiness probes.

## Scenario

An API legitimately needs about **90 seconds** to initialize. Its liveness check begins early and repeatedly fails because the process has not finished starting. The kubelet restarts it before startup can complete. The app enters a restart loop even though it might be healthy once initialized.

## Why startup probes help

A `startupProbe` gives a slow application a separate startup window. While startup has not succeeded, Kubernetes **does not execute liveness or readiness probes** for that container. After startup succeeds, the normal probes take over.

```text
Container starts
      |
      v
Startup probe checks /startup
      | (not yet successful: liveness/readiness held back)
      v
Startup succeeds
      |
      +--> Liveness: should process continue running?
      +--> Readiness: can it receive traffic?
```

## Example configuration

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  periodSeconds: 10
  failureThreshold: 12

livenessProbe:
  httpGet:
    path: /live
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 2
```

Here the startup budget is approximately **12 × 10 = 120 seconds**, subject to scheduling, probe timing, `timeoutSeconds`, and other runtime details. Configure enough margin for realistic worst-case startup, not only the average 90 seconds. Startup probe failures beyond its threshold cause the kubelet to restart the container.

The `/startup`, `/live`, and `/ready` endpoints must be implemented by the application; their behavior is not provided by Kubernetes automatically.

## Diagnosis

```bash
kubectl get pods -l app=payment-api
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
kubectl get deployment payment-api -o yaml
```

Look for `Unhealthy` probe events and restarts happening before the application reports startup complete. Check for application faults too: a startup probe should not simply hide a genuine crash or invalid configuration.

## Remediation

1. Measure startup time from application and deployment metrics.
2. Add/tune `startupProbe` with a realistic failure budget.
3. Keep liveness focused on recoverable application deadlocks, and readiness focused on traffic eligibility.
4. Review the Deployment change through GitOps.
5. Verify that new Pods start, become Ready, and stop restarting unexpectedly.

```bash
kubectl rollout status deployment/payment-api --timeout=180s
kubectl get pods -l app=payment-api -w
```

`--timeout=180s` is an illustrative observation timeout, not a mandatory setting.

## Operational lesson

**Startup protects slow initialization; liveness protects against unhealthy processes; readiness protects request routing.** Do not use liveness to enforce a startup deadline that healthy applications cannot meet.
