# 15 — Kustomize base/overlay configuration and relative-path error

**Type:** **Actually reproduced locally** in the `gcp-platform-gitops` repository. No Kubernetes cluster was running; no GCP deployment occurred.  
**Area:** GitOps, Kubernetes YAML generation, path troubleshooting.

## Goal

Use one shared NGINX Deployment and customize environment-specific settings without copying YAML into each environment.

```text
apps/demo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml            # added after the first successful build
│   └── kustomization.yaml
└── overlays/
    └── dev/
        └── kustomization.yaml
```

A **base** contains common resources. An **overlay** imports the base and applies environment-specific customizations. `kubectl kustomize` renders YAML locally; it does not install resources on a cluster.

## The error we observed

Command:

```bash
kubectl kustomize apps/demo/base
```

Output included:

```text
accumulating resources from '../../base'
evalsymlink failure on .../gcp-platform-gitops/apps/base
no such file or directory
```

**Root cause:** The base's `kustomization.yaml` incorrectly used `resources: [../../base]`. That path belongs in the **dev overlay**, not the base.

Kustomize resolves resource paths relative to the `kustomization.yaml` that declares them, not the terminal working directory.

From `apps/demo/base/`, `../../base` points to `apps/base/`, which did not exist.

## Correct files

### `apps/demo/base/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

For a production workload, pin an immutable image digest or controlled version rather than a floating tag.

### `apps/demo/base/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

If `service.yaml` has not been created yet, omit it until the file exists. The first successful lab build used only `deployment.yaml`.

### `apps/demo/base/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo
spec:
  type: ClusterIP
  selector:
    app: demo
  ports:
    - port: 80
      targetPort: 80
```

### `apps/demo/overlays/dev/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: demo-dev
replicas:
  - name: demo
    count: 2
```

## How we verified it (actual local outputs)

```bash
kubectl kustomize apps/demo/base
kubectl kustomize apps/demo/overlays/dev
```

The **base build** printed a Deployment named `demo`, with `replicas: 1` and no explicit namespace.

The **dev overlay build** printed the same Deployment customized with:

```yaml
metadata:
  name: demo
  namespace: demo-dev
spec:
  replicas: 2
```

Kustomize generated the overlay without editing the original `deployment.yaml`.

**Recorded scope:** The outputs in the actual session showed only the Deployment. A Service manifest was proposed afterward, but its rendered output was not shown in the session. When `service.yaml` is added to the base Kustomization, rendering should produce both Deployment and Service.

## Follow-up: namespace creation

Setting `namespace: demo-dev` changes the namespace field of namespaced resources; it **does not create the Namespace object**. For GitOps, either add a Namespace manifest and include it in the desired state or deliberately configure Argo CD namespace creation. Avoid leaving a hidden manual prerequisite.

Example manifest:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-dev
```

If using a Namespace manifest, manage it from an explicit, well-defined path and avoid duplicate ownership between Terraform and Argo CD.

## Debugging checklist

```bash
find apps/demo -type f -maxdepth 5 | sort
cat apps/demo/base/kustomization.yaml
cat apps/demo/overlays/dev/kustomization.yaml
kubectl kustomize apps/demo/base
kubectl kustomize apps/demo/overlays/dev
```

When `accumulating resources` appears, check the folder holding the Kustomization, its `resources` paths, capitalization, and whether referenced files exist.

## Operational lesson

**Kustomize renders a desired YAML configuration; Argo CD or `kubectl apply -k` later applies it.** Separate common manifests from environment-specific differences, and validate output locally before deploying.
