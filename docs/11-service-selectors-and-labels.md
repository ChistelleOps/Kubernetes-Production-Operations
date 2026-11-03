# 11 — Service selector mismatch: Pods are healthy but traffic fails

**Type:** Real-world troubleshooting scenario discussed in the lab, **not deployed/tested on GKE**.  
**Area:** Kubernetes networking, Service discovery, labels, EndpointSlices.

## Symptom

A Deployment has healthy Pods, but callers cannot reach the application through its Kubernetes Service. The Service exists and has a ClusterIP. That **does not** guarantee that it has usable backends.

Example Pods:

```yaml
metadata:
  labels:
    app: payment-api
```

But the Service was configured with:

```yaml
spec:
  selector:
    app: payments
```

`payment-api` and `payments` are not equal. Unless **other** Pods match `app=payments`, the Service has no selected backends.

## Why this happens

A Service selector matches **Pod labels**; it does not point to a Deployment object or a specific Pod name. A Deployment labels the Pod templates it creates. The EndpointSlice controller tracks addresses and conditions for Pods selected by the Service.

```text
Service selector app=payments
             |
             +-- Pod A app=payment-api  (doesn't match)
             +-- Pod B app=payment-api  (doesn't match)
             +-- Pod C app=payment-api  (doesn't match)
             |
             +-- no matching backends -> requests fail
```

A replacement Pod commonly gets a different name and IP. Labels provide a stable way to discover it. A Service can match Pods from any controller, not only one Deployment.

## Investigation — commands and interpretation

Set the namespace explicitly if the resources are not in `default`:

```bash
NS=default
kubectl -n "$NS" get svc payment-service -o wide
kubectl -n "$NS" describe svc payment-service
```

Look at `Selector` and the displayed `Endpoints` field. `Endpoints: <none>` is a clue, not the complete diagnosis.

```bash
kubectl -n "$NS" get pods --show-labels
kubectl -n "$NS" get pods -l app=payments -o wide
kubectl -n "$NS" get pods -l app=payment-api -o wide
```

If the first selector finds nothing but the second finds healthy Pods, the label mismatch is confirmed.

```bash
kubectl -n "$NS" get endpointslices \
  -l kubernetes.io/service-name=payment-service -o yaml
```

In the output, inspect `endpoints`, addresses, and readiness conditions. Newer clusters use EndpointSlices; do not rely exclusively on the older Endpoints API.

## Resolution

Correct the Service **in the source of truth**, which in our platform will be the GitOps repository:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-api
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

Review the Git change and let Argo CD reconcile it when the cluster is available. For an emergency manual correction, follow the team's change-control process; afterward reconcile Git to prevent the old selector returning.

## Verify

```bash
kubectl -n "$NS" describe svc payment-service
kubectl -n "$NS" get endpointslices \
  -l kubernetes.io/service-name=payment-service -o yaml
kubectl -n "$NS" get pods -l app=payment-api
```

Confirm the Service selects the intended Pods, EndpointSlices contain **ready** backends, and an in-cluster request to `http://payment-service` succeeds.

## Next checks if labels match but requests still fail

- Are the matching Pods actually **Ready**? See [12 — Running but Not Ready](12-running-but-not-ready.md).
- Does `targetPort` match the application listener? `port` is the Service-facing port; `targetPort` is the backend Pod port.
- Do the Service and selected Pods share the same namespace? A Service selector selects Pods in its own namespace.
- Are NetworkPolicies, application errors, or DNS issues involved?

## Operational lesson

**Healthy Pods + existing Service does not imply healthy Service backends.** Trace Service selector → Pod labels → EndpointSlices → readiness → port mapping → end-to-end traffic.
