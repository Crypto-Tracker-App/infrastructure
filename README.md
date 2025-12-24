# Infrastructure

Kubernetes manifests and Kustomize overlays for the Crypto-Tracker-App cluster infrastructure.

This repository contains base manifests and environment overlays. It includes Namespace resources for the following environments:

- `prod`
- `test`
- `dev`

Apply the namespaces directly:

\`\`\`powershell
kubectl apply -f k8s/base/namespace.yaml
\`\`\`

Repository layout

- `k8s/base/` — base manifests (namespaces, common resources)

Notes

- Namespaces are defined in `k8s/base/namespace.yaml`.
- Adjust RBAC, resource quotas, and network policies per environment in overlays.
