# 13 — Readiness versus liveness during a PostgreSQL outage

**Type:** Hypothetical production scenario discussed for learning; **not tested in the lab**.  
**Area:** Health checks, external dependencies, avoiding restart loops.

## Scenario

A payment API process is running. PostgreSQL becomes temporarily unavailable. The application's readiness endpoint requires a working database connection, but the process can continue running and reconnect when PostgreSQL returns.

**Question:** Should Kubernetes restart every application container merely because the database is down?

Usually **no**. Restarting healthy clients will not restore the database and may create cascading load, especially if every replica restarts repeatedly.

## What each probe is for

| Probe | Main question | Failure behavior |
| --- | --- | --- |
| Readiness | Can this container serve normal traffic *now*? | Container is marked Not Ready; Pod normally excluded from Service endpoints |
| Liveness | Is this container stuck or irrecoverably unhealthy? | After configured failures, kubelet restarts the container |
| Startup | Has a slow-starting process completed startup? | Suppresses liveness/readiness until it succeeds; repeated failure causes restart |

**Probe behavior depends on implementation.** A database outage does not inherently make a readiness probe fail: the application must expose a readiness signal that includes that dependency when appropriate.

## Example health endpoints

- `/live`: checks that the application process/event loop is functioning. It does **not** query PostgreSQL for every probe.
- `/ready`: checks that the application can serve requests, potentially including essential database readiness.

```yaml
containers:
  - name: payment-api
    image: example.invalid/payment-api:v1  # illustrative placeholder, not a usable image
    ports:
      - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
    livenessProbe:
      httpGet:
        path: /live
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
```

This manifest snippet illustrates probe configuration; the endpoint implementation is part of the application code. The image URL is deliberately a placeholder and should not be deployed.

## Expected sequence

1. PostgreSQL becomes unavailable.
2. `/ready` returns an unsuccessful response if database availability is a readiness requirement.
3. After the configured probe threshold, the container becomes Not Ready.
4. The Service normally stops sending it new traffic.
5. `/live` can continue passing because the process remains healthy and is capable of recovery.
6. When PostgreSQL recovers and `/ready` succeeds, the Pod becomes eligible for traffic again.

If **all** replicas are Not Ready, the Service has no healthy normal backend; readiness protects against sending traffic to unready instances but cannot itself restore a database outage.

## Investigation

```bash
kubectl get pods -l app=payment-api
kubectl describe pod <pod-name>
kubectl logs <pod-name> --tail=100
kubectl get deployment payment-api -o yaml
kubectl get endpointslices -l kubernetes.io/service-name=payment-api -o yaml
```

Check Kubernetes events for readiness/liveness failures and correlate with database alerts, latency, and error metrics.

## Common mistakes

- **Using `/ready` as `/live`** even though `/ready` fails when an external service is down: may cause avoidable restarts.
- **Making readiness too strict**: sometimes nonessential external dependencies shouldn't make *all* endpoints disappear.
- **Making liveness too permissive**: an irrecoverably deadlocked process might never restart.
- **Assuming readiness restarts containers**: it doesn't; liveness or startup probe failures can.

## Operational lesson

Readiness controls **traffic eligibility**; liveness controls **restart/recovery of a container**. Choose the probe contract based on what the application can actually recover from.
