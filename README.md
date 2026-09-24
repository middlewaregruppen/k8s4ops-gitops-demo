# k8s4ops GitOps demo

Source of truth for the ArgoCD lab in the **k8s4ops** course. It is deliberately
tiny: a hardened Deployment, a Service, and a ConfigMap, with a second overlay
that changes only the replica count. Nothing here needs to be built, pushed, or
configured — ArgoCD reads it straight from Git.

This repository is public course material. It exists so that students can point
an `Application` at a real Git repository without needing a GitHub account of
their own, which the course cannot assume.

## Layout

```
apps/
  guestbook/            base: Deployment + Service + ConfigMap
  guestbook-scaled/     overlay: same app, 4 replicas instead of 2
```

## Using it from ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/middlewaregruppen/k8s4ops-gitops-demo.git
    targetRevision: main
    path: apps/guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: gitops-demo
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Rendered locally with:

```bash
kubectl kustomize apps/guestbook
```

## Where the interesting states are

| Change in Git | What ArgoCD reports |
| --- | --- |
| nothing | `Synced` / `Healthy` |
| `replicas: 2` → `4` in the base | `OutOfSync` until synced |
| switch `path` to `apps/guestbook-scaled` | `OutOfSync`, and 4 Pods after sync |
| edit `ConfigMap/greeting` live in the cluster | `OutOfSync` |

The last row is the one worth watching: the cluster can be edited directly, and
with self-healing enabled ArgoCD puts it back. Git is the only thing that gets
to decide what the application looks like.

## Maintenance

The `replicas` count and the image tag are the intended knobs — an instructor can
push a one-line change during a session and students watch ArgoCD notice it,
without anyone needing write access. Keep the app small: it is a teaching
example, not a service.
