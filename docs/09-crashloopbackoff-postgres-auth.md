# 09 — `CrashLoopBackOff` after PostgreSQL authentication failure

**Category:** Application startup / credentials / dependency failures  
**Status:** Hypothetical incident scenario — **not reproduced** in the GKE lab

> ⚠️ **Not verified in the lab.** This is a reasoned scenario. Treat commands and conclusions as a starting point and validate them in your environment.

## Scenario and evidence

A payment application becomes unavailable after a deployment:

```text
payment-api-7d8f9c-abc12   0/1   CrashLoopBackOff   6
```

The previous container's logs include:

```text
Starting payment-api...
Connecting to PostgreSQL...
FATAL: password authentication failed for user "payment_app"
Application shutting down.
```

**Immediate cause:** the application exits because its PostgreSQL login is rejected. Kubelet retries restarting the container and progressively backs off, producing `CrashLoopBackOff`.

**Root cause still unknown:** a wrong Secret, changed password, wrong username/database endpoint, or a recent app/config release may be responsible. The auth error suggests that a PostgreSQL endpoint responded; it does not prove the correct endpoint was reached.

## Triage and evidence

```bash
kubectl get pods -n <namespace>
kubectl describe pod -n <namespace> <pod-name>
kubectl logs -n <namespace> <pod-name> -c <container-name> --previous --tail=100
kubectl describe deployment -n <namespace> payment-api
kubectl rollout history deployment/payment-api -n <namespace>
```

- Check **last termination state**, Pod events, restart count, and first failure time.
- Read prior container logs, since current logs may contain only the newest failed startup.
- Inspect the Deployment's **image**, environment variable names, `secretKeyRef` mappings, ConfigMaps, and database host settings. These describe **where** credentials come from without revealing their values.
- Correlate incident start with deployments, database maintenance, Secret rotation, and external-secret synchronization.

Inspect Secret metadata and key names without decoding its values:

```bash
kubectl get secret -n <namespace>
kubectl describe secret -n <namespace> <secret-name>
```

A successful Secret lookup does **not** prove that its values match the database. Validate credentials securely through the authorized secret-management and database-administration workflow. Do **not** paste plaintext passwords, decode Secret data into tickets, or log connection strings containing credentials.

## Prioritize recovery without guessing

1. Determine if a **known-good previous Deployment revision** still uses valid credentials. If the latest release introduced a bad Secret reference or connection setting, a rollback may help:

   ```bash
   kubectl rollout undo deployment/payment-api -n <namespace>
   kubectl rollout status deployment/payment-api -n <namespace> --timeout=120s
   ```

2. If the password was rotated separately, a Deployment rollback **will not restore the previous Kubernetes Secret** and may not help. Coordinate the credential update instead (see [10 — Secret rotation](10-secret-rotation-env-vars.md)).
3. Verify application readiness, successful DB authentication, critical transactions, and incident metrics, not just that containers are `Running`.

## Prevention and observations

Use approved Secret-management workflows, coordinated password rotation, startup/readiness checks, change history, and alerts on auth errors. Keep sensitive data out of logs. Treat the *auth failure* as observed in this **scenario**, not as proof that the cluster network or Kubernetes Service is broken.
