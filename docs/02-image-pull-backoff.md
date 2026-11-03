# 02 — Deployment update with an invalid image tag

**Category:** Container image / rollout  
**Environment:** GKE Standard; NGINX Deployment `web`  
**Status:** Failure deliberately injected in the lab

## The change

```bash
kubectl set image deployment/web nginx=nginx:this-tag-does-not-exist
```

Here, `deployment/web` identifies the Deployment; `nginx=` targets the container named `nginx`; the right side is the proposed image reference. The invalid tag was intentional. This command updates the **Deployment Pod template**, not the image inside an existing running container.

## What happens behind the scenes

1. `kubectl` sends the updated Deployment to the Kubernetes API server.
2. The Deployment controller sees a new Pod template and creates a **new ReplicaSet**.
3. The ReplicaSet controller creates replacement Pods for the new revision.
4. The scheduler assigns a new Pod to a node if resources permit.
5. The **kubelet** instructs containerd to pull the image.
6. The invalid tag cannot be retrieved, so the container cannot start. Typical states are `ErrImagePull` and subsequently `ImagePullBackOff` as Kubernetes delays retries.

A Pod with `ImagePullBackOff` may have been scheduled successfully: this is distinct from a Pod stuck `Pending` with `FailedScheduling` events.

## Investigation: commands and purpose

```bash
kubectl get deployment web
# Compare DESIRED/READY/AVAILABLE where shown.

kubectl get pods -o wide
# Identify the new non-running Pod and its node assignment.

kubectl describe pod <new-pod-name>
# Inspect Image, State, and Events; look for image not found,
# registry authorization failure, or network-related pull errors.

kubectl rollout status deployment/web --timeout=30s
# Determine whether the new revision becomes available.

kubectl rollout history deployment/web
# See revisions; use descriptions and revision history for context.
```

**Interpretation:** Do not decide the cause from `ImagePullBackOff` alone. Events distinguish a nonexistent tag from authentication errors, registry outages, bad image names, or other pull issues. Our injected change targeted a nonexistent tag; this was the intended fault.

## Why old Pods may still be running

Deployments commonly use the **RollingUpdate** strategy. Kubernetes can create new Pods while retaining old healthy Pods within the configured `maxSurge`/`maxUnavailable` limits. A bad new revision can stall while old replicas continue serving. This protects availability but is not an automatic rollback.

## Recovery

If the preceding revision is known to be working:

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web --timeout=60s
kubectl get pods -o wide
```

`rollout undo` returns the Deployment's Pod template to an earlier revision. The control plane then reconciles toward that template. Confirm the number of **Ready/Available** replicas, not just that one Pod says Running.

In GitOps environments, also correct the image/tag in Git. Otherwise the GitOps controller or a future deployment may reintroduce the broken image.

## Production considerations

- Prefer immutable image digests (for example `kubectl set image deployment/web nginx=nginx@sha256:<digest>`) or carefully managed release tags.
- Validate image publication and registry access before promoting a release.
- Use readiness probes and sensible rollout parameters.
- Monitor Deployment rollout status and alert on unavailable replicas.
- Do not paste registry credentials or pull-secret contents into issue reports.

## Related

- [04 — Insufficient cpu](04-insufficient-cpu.md): another way a rollout stalls while old Pods keep serving.

## Key distinction

`ImagePullBackOff`: **image retrieval/startup problem** after a Pod was created.  
`FailedScheduling`: **placement problem** before a Pod can run on any node.
