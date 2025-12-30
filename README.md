# Ingress & Service Endpoints Guide

This document explains how to create and expose endpoints through the Nginx Ingress Controller.

## Overview

Services are exposed through two Nginx Ingress resources at `http://20.251.246.218/`:

1. **frontend-ingress** - Serves the frontend application (no path rewriting)
2. **api-ingress** - Routes API requests to backend services (with path rewriting)

## Service Routing

| Service | Public URL Pattern | Internal Port | Example |
|---------|------------------|----------------|---------|
| **frontend** | `http://<IP>/` or `http://<IP>/*` | 80 | `http://<IP>/`, `http://<IP>/assets/index.js` |
| **pricing-service** | `http://<IP>/pricing-service/api/...` | 12000 | `http://<IP>/pricing-service/api/top-coins` |
| **user-service** | `http://<IP>/user-service/api/...` | 5000 | `http://<IP>/user-service/api/users` |

## Important Routing Rules

⚠️ **API Endpoints Only**: Only paths matching `/api/*` are publicly accessible for pricing-service and user-service.

- ✅ `http://<IP>/user-service/api/users/login` → Routes to `http://user-service:5000/api/users/login`
- ✅ `http://<IP>/assets/index.js` → Routes to frontend (static assets work)
- ❌ `http://<IP>/user-service/health` → Returns 404 (not exposed)
- ❌ `http://<IP>/pricing-service/metrics` → Returns 404 (not exposed)

## How Path Rewriting Works

### Frontend (No Rewriting)
Frontend requests pass through unchanged:
```
Request:  /assets/index.js
           ↓
Sent to: frontend:80/assets/index.js
```

### API Services (With Rewriting)
The API Ingress uses regex-based path rewriting:

```
Request:  /pricing-service/api/top-coins
           └───────┬───────┘└─┬─┘└───┬───┘
                   │          │      │
            Matched prefix   $1     $2
          /pricing-service/api(/|$)(.*)

Rewrite Rule: /api/$2
           ↓
Result: /api/top-coins
           ↓
Sent to: pricing-service:12000/api/top-coins
```

## Creating New Endpoints

### For API Endpoints

Your service should:
1. Listen on the port defined in the Ingress (e.g., 5000 for user-service)
2. Define endpoints under `/api/*` path (e.g., `/api/users`, `/api/login`)
3. **Do NOT** define health checks under `/api/` (they should not be publicly accessible)

### Example (User Service)

```python
# ✅ Public endpoint - accessible via ingress
@app.route('/api/users', methods=['GET'])
def get_users():
    return jsonify(users)

# ❌ NOT publicly accessible - define elsewhere
@app.route('/health', methods=['GET'])
def health():
    return jsonify(status='ok')
```

## Adding a New Service

To add a new API service to the Ingress:

1. Update `k8s/base/ingress.yaml` in the `api-ingress` section:
   ```yaml
   - path: /my-service/api(/|$)(.*)
     pathType: ImplementationSpecific
     backend:
       service:
         name: my-service
         port:
           number: 8080
   ```

2. Ensure your service is exposed as a ClusterIP service in Kubernetes

3. Apply the changes:
   ```bash
   kubectl apply -f k8s/base/ingress.yaml
   ```

4. Test the endpoint:
   ```bash
   curl http://<INGRESS_IP>/my-service/api/endpoint
   ```

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| 404 Not Found | Path doesn't match ingress rules | Check that path matches `/service-name/api/*` pattern |
| Connection refused | Service not running | `kubectl get pods` and check service status |
| Service not found | Service name or port incorrect | Verify service name and port in `kubectl get svc` |
| Ingress not assigned IP | Nginx controller not ready | `kubectl get pods -n ingress-nginx` |
| Static assets 404 | Wrong ingress handling assets | Ensure frontend-ingress has `/` path with Prefix type |

## Ingress Architecture

```
                    ┌─────────────────────────────────────┐
                    │         Nginx Ingress Controller    │
                    │           (External IP)             │
                    └──────────────┬──────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
    ┌─────────▼─────────┐ ┌───────▼───────┐ ┌─────────▼─────────┐
    │  frontend-ingress │ │  api-ingress  │ │    api-ingress    │
    │    path: /        │ │ /pricing-svc  │ │   /user-svc       │
    │   (no rewrite)    │ │  (rewrite)    │ │    (rewrite)      │
    └─────────┬─────────┘ └───────┬───────┘ └─────────┬─────────┘
              │                   │                   │
    ┌─────────▼─────────┐ ┌───────▼───────┐ ┌─────────▼─────────┐
    │   frontend:80     │ │ pricing:12000 │ │   user:5000       │
    └───────────────────┘ └───────────────┘ └───────────────────┘
```

## Current Ingress Configuration

See [ingress.yaml](./ingress.yaml) for the full configuration.


# Infrastructure

Kubernetes manifests and Kustomize overlays for the Crypto-Tracker-App cluster infrastructure.

This repository contains base manifests and environment overlays. It includes Namespace resources for the following environments:

- `prod`
- `test`
- `dev`
- `logging`

Apply the namespaces directly:

\`\`\`powershell
kubectl apply -f k8s/base/namespace.yaml
\`\`\`

Repository layout

- `k8s/base/` — base manifests (namespaces, common resources)

Notes

- Namespaces are defined in `k8s/base/namespace.yaml`.
- Adjust RBAC, resource quotas, and network policies per environment in overlays.
