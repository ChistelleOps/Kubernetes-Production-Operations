# 10 — Database password rotation: Secret updated but Pods still use old credentials

**Category:** Kubernetes Secrets / configuration refresh / operational safety  
**Status:** Hypothetical production scenario — **not reproduced** in the GKE lab

> ⚠️ **Not verified in the lab.** This is a reasoned scenario. Treat commands and conclusions as a starting point and validate them in your environment.

## Scenario

A payment API authenticates to PostgreSQL with a password supplied by a Kubernetes Secret via an **environment variable**. The database team rotated the password, but the Kubernetes Secret still contains the old value. No application release occurred. New authentication attempts fail.

Conceptual flow:

```text
PostgreSQL expects NEW password
         !=
Kubernetes Secret has OLD password
         ->
Container environment variable has OLD password
         ->
PostgreSQL rejects login
```

## Key behavior: env vars versus mounted Secret files

- **Secret referenced as an environment variable:** the container receives the value **when it starts**. Updating the Secret does not change an already-running container's environment. Newly started containers can read the current value.
- **Secret mounted as a file:** kubelet normally refreshes projected Secret volume contents eventually, subject to synchronization delay; **`subPath` mounts do not receive automatic updates**. Even when the file contents update, the application must reread or reload them. A process caching the password may need an application-specific reload or restart.

This is why **updating the Secret alone may not restore service**.

## Investigate safely

```bash
kubectl describe deployment -n <namespace> payment-api
kubectl get secret -n <namespace>
kubectl describe secret -n <namespace> <secret-name>
kubectl logs -n <namespace> deployment/payment-api --tail=50
```

Check which Secret and key the Deployment references; whether an External Secrets controller (if used) has synchronized the latest version; and when the database password changed. `kubectl describe secret` displays key names and lengths without decoding the data. Do not print, share, or commit Secret values.

## Restore service: production-safe order

1. **Coordinate with the database and secret owners** to confirm the active database credentials and permitted rotation process. Ideally, use an overlap period or dual credentials to avoid breaking old Pods.
2. **Update/synchronize the correct Kubernetes Secret** using the authorized secret-management workflow (for example, external secret manager → External Secrets Operator). Verify synchronization **without exposing the value**.
3. Because credentials are injected via **environment variables**, start new containers with the current Secret:

   ```bash
   kubectl rollout restart deployment/payment-api -n <namespace>
   kubectl rollout status deployment/payment-api -n <namespace> --timeout=120s
   kubectl get pods -n <namespace>
   ```

4. **Verify the application actually authenticates successfully**, readiness checks pass, and failures decline. If rollout fails, review events and previous logs instead of restarting repeatedly.

**Caution:** If the Secret is still stale when you restart, new Pods will start with the **same wrong password**. And if every replica is already failing, a rolling restart cannot prevent the existing outage.

## Why not simply roll back?

`kubectl rollout undo` changes the Deployment's Pod template to an earlier revision. It does not automatically undo out-of-band changes to a Secret or database password. Use rollback for a bad Deployment change, not as the default fix for a credential rotation mismatch.

## Verification checklist

- Secret reference and key match the intended workload.
- Approved secret system and Kubernetes Secret are synchronized.
- Fresh Pods were created and became Ready.
- Authentication and transaction checks succeed.
- No credential value appeared in logs, Git, terminal history, or incident comments.

## Automating the restart

To avoid a manual `rollout restart` after every rotation, common options are a controller such as [Stakater Reloader](https://github.com/stakater/Reloader) that restarts workloads when a referenced Secret changes, or a Pod-template annotation containing a hash of the Secret (for example via Helm or Kustomize) so that a change triggers a normal rolling update. Evaluate either against your change-control policy.

## Prevention

Use coordinated rotations, graceful credential overlap when supported, automated synchronization, controlled rollout/reload mechanisms, and alerting on authentication failures and Secret-sync errors. Keep credentials out of source control.
