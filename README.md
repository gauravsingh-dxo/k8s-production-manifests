# k8s-production-manifests

Kubernetes manifests for a small API service, written the way I'd actually deploy something to production — not the stripped-down "hello world" YAML most tutorials stop at. Uses Kustomize base + overlays to handle dev/prod differences without duplicating manifests.

## What this demonstrates

- Resource `requests` and `limits` set explicitly (not omitted, which is the most common gap I see in manifests that "work in a demo")
- Liveness and readiness probes pointed at a real health endpoint, with different `initialDelaySeconds` for each — readiness should fire faster than liveness
- Pod-level and container-level `securityContext`: non-root user, dropped Linux capabilities, read-only root filesystem
- Horizontal Pod Autoscaler with separate CPU/memory targets and a `scaleDown` stabilization window (prevents flapping when traffic is spiky)
- Kustomize `base/` + `overlays/` structure so dev and prod share 90% of their config and diverge only where they need to

## Architecture

```
base/                    (shared manifests - never deployed directly)
├── deployment.yaml
├── service.yaml
├── configmap.yaml
├── hpa.yaml
└── ingress.yaml

overlays/dev/             overlays/prod/
├── kustomization.yaml     ├── kustomization.yaml
└── (1 replica,             └── (4 replicas via
     debug logging)              replica-patch.yaml)
```

## Prerequisites

- A running Kubernetes cluster (kind/minikube for local testing, or any managed cluster)
- `kubectl` >= 1.28 (built-in Kustomize support)
- An ingress-nginx controller and cert-manager installed if you want the Ingress resource to actually issue certs

## Usage

```bash
# Preview the rendered manifests for an environment
kubectl kustomize overlays/dev

# Apply directly
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod
```

## Design decisions

**Why Kustomize instead of Helm?** For a single service with a handful of environment differences (replica count, log level), Kustomize's patch-based approach is less machinery than templating a full Helm chart. I'd reach for Helm once there are multiple services sharing a chart, or the manifests need to be distributed to other teams as a package.

**Why `readOnlyRootFilesystem: true`?** It's a small line that catches a real class of problems — if the container tries to write anywhere outside a mounted volume (including from a compromised dependency), it fails loudly instead of silently succeeding. Any legitimate writable path (temp files, caches) should get an explicit `emptyDir` volume mount, not a writable root filesystem.

**What I'd add before this is a real production deployment:** NetworkPolicies restricting pod-to-pod traffic, PodDisruptionBudgets so voluntary node drains don't take out all replicas at once, and resource quotas at the namespace level.
