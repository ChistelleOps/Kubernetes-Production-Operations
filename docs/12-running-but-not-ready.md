# 12 — Pod is `Running` but `0/1 Ready`: why a Service may have no healthy endpoints

**Type:** Real-world scenario discussed for learning; **not reproduced on GKE**.  
**Area:** Pod conditions, readiness, Service EndpointSlices.

## Symptom

```text
NAME                    READY   STATUS
payment-api-abc123      0/1     Running
payment-api-def456      0/1     Running
```

These Pods are in the **Running phase**, but zero of the one application container in each Pod is Ready.

- `0/1` is a count of **Ready containers**, not a count of Running Pods.
- `Running` is a Pod phase; it does not mean the application is usable.
- A container may run successfully while a readiness probe fails, or while another readiness condition is not satisfied.

## Why this matters to the Service

Assume the Service selector matches `app=payment-api`. EndpointSlices may still show these Pods with `conditions.ready: false`. Under normal Service behavior, non-ready endpoints are excluded from regular traffic.

```text
Service selector: app=payment-api
    |
    +-- Pod A: Running, Ready=False -> not eligible for normal traffic
    +-- Pod B: Running, Ready=False -> not eligible for normal traffic
    |
    +-- no ready backends -> connection/application availability problem
```

A matching label is necessary for selector-based routing, but **readiness determines whether a matching endpoint is usable**. There are special Service settings and conditions (for example `publishNotReadyAddresses`) that alter normal behavior; investigate the actual Service configuration before generalizing.

## Step-by-step diagnosis

### 1. Inspect Pod conditions and events

```bash
kubectl describe pod payment-api-abc123
kubectl get pod payment-api-abc123 -o yaml
```

Look at `Ready`, `ContainersReady`, container state, and `Events`. Example clue:

```text
Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 503
```

This would show the readiness endpoint responded with an unsuccessful status; it does **not** by itself establish why.

### 2. Read application logs

```bash
kubectl logs payment-api-abc123 --tail=100
```

For a container that restarted, examine the previous instance as appropriate:

```bash
kubectl logs payment-api-abc123 --previous
```

Look for startup failures, database dependency problems, failed health endpoints, configuration errors, or timeouts.

### 3. Inspect the readiness configuration

```bash
kubectl describe deployment payment-api
kubectl get deployment payment-api -o yaml
```

Confirm `readinessProbe` protocol, path, port, timeouts, thresholds, and whether the app serves the expected endpoint. A readiness probe that always fails because of a wrong path or port is a configuration defect; a probe failing because a required dependency is unavailable may reflect real application unavailability.

### 4. Inspect Service EndpointSlices

```bash
kubectl get svc payment-api -o yaml
kubectl get endpointslices \
  -l kubernetes.io/service-name=payment-api -o yaml
```

Compare selectors, selected Pod labels, and endpoint `ready` conditions.

## Partial failure: 2 Ready out of 3

```text
payment-api-abc123     1/1   Running
payment-api-def456     0/1   Running
payment-api-ghi789     1/1   Running
```

Under normal Kubernetes Service behavior, **only the two Ready endpoints** receive new Service traffic. Kubernetes does not normally route a share to the Not Ready Pod merely because it has a matching label. Nevertheless, remaining healthy Pods can become overloaded, or other dependencies can still generate errors.

## How to fix and verify

Fix the diagnosed issue rather than just restarting Pods: correct the probe configuration, restore a required dependency, or repair the application. Then verify:

```bash
kubectl get pods -l app=payment-api
kubectl rollout status deployment/payment-api
kubectl get endpointslices \
  -l kubernetes.io/service-name=payment-api -o yaml
```

Test real application traffic and review error rates. For Deployments, `Available` can also depend on `minReadySeconds`, not just current readiness.

## Operational lesson

**Running ≠ Ready ≠ Available.** Do not troubleshoot Service selectors alone when selected Pods are Not Ready.
