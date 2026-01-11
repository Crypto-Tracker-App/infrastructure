# Crypto Tracker Application - Infrastructure Documentation

## Overview

This directory contains the Kubernetes infrastructure configuration for the **Crypto Tracker Application**, a microservices-based cryptocurrency monitoring platform.

## Architecture Summary

The application follows a **microservices architecture** with the following components:

- **Frontend**: React-based web interface
- **User Service**: Authentication and user management (JWT-based)
- **Pricing Service**: Real-time cryptocurrency price data and market analysis
- **Portfolio Service**: Portfolio tracking and management
- **Alert Service**: Notification and alerting system

Each microservice is backed by a PostgreSQL database and communicates through a centralized NGINX Ingress controller.

## CI/CD Pipeline

Each service maintains its own independent CI/CD pipeline to enable.


## Directory Structure

```
k8s/
├── base/                          # Base configurations (shared across all environments)
│   ├── ingress.yaml              # Ingress routing rules
│
├── frontend/                      # Frontend application
│   ├── frontend-config.yaml       # ConfigMap for frontend configuration
│   ├── frontend-deployment.yaml   # Frontend deployment specification
│   ├── frontend-service.yaml      # Frontend service definition
│   └── kustomization.yaml         # Frontend Kustomize configuration
│
├── user-service/                  # User management microservice
│   ├── userservice-configmap.yaml # User service configuration
│   ├── userservice-deployment.yaml # User service deployment
│   ├── userservice-postgres.yaml   # PostgreSQL database for user service
│   ├── userservice-service.yaml    # User service network service
│   └── kustomization.yaml          # User service Kustomize configuration
│
├── pricing-service/               # Cryptocurrency pricing microservice
│   ├── pricingservice-configmap.yaml   # Pricing service configuration
│   ├── pricingservice-deployment.yaml  # Pricing service deployment
│   ├── pricingservice-jobs.yaml        # Batch jobs for price updates
│   ├── pricingservice-postgres.yaml    # PostgreSQL database for pricing service
│   ├── pricingservice-service.yaml     # Pricing service network service
│   └── kustomization.yaml              # Pricing service Kustomize configuration
│
├── portfolio-service/             # Portfolio management microservice
│   ├── portfolioservice-configmap.yaml  # Portfolio service configuration
│   ├── portfolioservice-deployment.yaml # Portfolio service deployment
│   ├── portfolioservice-postgres.yaml   # PostgreSQL database for portfolio service
│   ├── portfolioservice-service.yaml    # Portfolio service network service
│   └── kustomization.yaml               # Portfolio service Kustomize configuration
│
├── alert-service/                 # Alert management microservice
│   ├── alertservice-configmap.yaml     # Alert service configuration
│   ├── alertservice-deployment.yaml    # Alert service deployment
│   ├── alertservice-postgres.yaml      # PostgreSQL database for alert service
│   ├── alertservice-service.yaml       # Alert service network service
│   └── kustomization.yaml              # Alert service Kustomize configuration
│
└── overlays/                       # Environment-specific overrides
    └── prod/                       # Production environment
        ├── alert-service/          # Production alert service overrides
        │   └── kustomization.yaml
        ├── frontend/               # Production frontend overrides
        │   └── kustomization.yaml
        ├── portfolio-service/      # Production portfolio service overrides
        │   └── kustomization.yaml
        ├── pricing-service/        # Production pricing service overrides
        │   └── kustomization.yaml
        └── user-service/           # Production user service overrides
            └── kustomization.yaml
```


#### `base/ingress.yaml`
Implements NGINX Ingress controller for external traffic routing, that redirects HTTP traffic to HTTPS:

1. **Frontend Ingress**: Routes HTTPS requests to `/` path to the frontend service
   - Uses try-files rewrite rule to support SPA (Single Page Application) routing
   - Serves static content on port 80

2. **API Ingress**: Routes microservice API requests with path-based routing
   - `/pricing-service/api/*` → pricing-service (port 12000)
   - `/user-service/api/*` → user-service (port 5000)
   - `/portfolio-service/api/*` → portfolio-service (port 5000)
   - `/alert-service/api/*` → alert-service (port 5000)

Each route uses URL rewriting to strip the service prefix from the request path.

### Microservice Configuration Pattern

Each microservice directory follows a consistent structure:

#### `*-configmap.yaml`
ConfigMaps store non-sensitive configuration data as environment variables:
- Database connection details (host, port, database name)
- Service port numbers
- Application logging levels
- Feature flags

#### `*-deployment.yaml`
Kubernetes Deployment specifications define:
- Pod replicas (typically 1 for single-node clusters)
- Container image references from Azure Container Registry
- Environment variables (from ConfigMaps and Secrets)
- Resource requests and limits
- Init containers for database initialization (wait-for-db)
- Volume mounts for persistent data
- Health checks (liveness and readiness probes)

#### `*-postgres.yaml`
StatefulSet definitions for PostgreSQL databases:
- Persistent Volume Claims (PVCs) for data storage
- Environment variables for database initialization
- Service definitions for database connectivity
- Database user and password secrets
- Initialization scripts (if applicable)

#### `*-service.yaml`
Kubernetes Service definitions for:
- ClusterIP services (internal communication between services)
- Service port mappings to container ports
- Selectors matching deployment pods
- DNS names for service discovery

#### `kustomization.yaml`
Kustomize configuration files that:
- Reference resource YAML files to include
- Define ConfigMap and Secret generators
- Apply common labels and name prefixes
- Configure image references
- Include base resources or other kustomizations

### Overlay Configuration

#### `overlays/prod/*/kustomization.yaml`
Production environment overrides:
- Override replica counts for high availability
- Adjust resource limits and requests for production workloads
- Apply production-specific images and versions
- Configure production logging and monitoring
- Set production-specific environment variables

## How It Works

### Deployment Flow

1. **Base Resources**: Common configurations in `base/` define the foundation
   - Ingress controller routes external traffic

2. **Service Definitions**: Each microservice includes:
   - ConfigMap: Service configuration
   - Deployment: Pod and container specifications
   - Service: Network exposure
   - Database: Persistent data storage

3. **Kustomize Composition**: 
   - Base `kustomization.yaml` files aggregate resources
   - Overlays apply environment-specific customizations
   - Build command: `kubectl apply -k overlays/prod/`

4. **Container Images**: All containers are pulled from Azure Container Registry (ACR)
   - Registry name: `cryptotracker`
   - Images are referenced with full paths in deployments

### Request Flow

```
Client Request
    ↓
NGINX Ingress (frontend-ingress / api-ingress)
    ↓
Frontend Service / Microservice (user, pricing, portfolio, alert)
    ↓
Application Pod (Flask microservice)
    ↓
PostgreSQL Database (persistent storage)
```

## Deployment Commands

### Apply base configuration:
```bash
kubectl apply -k k8s/base/
```

### Apply production overlay:
```bash
kubectl apply -k k8s/overlays/prod/
```

### View deployed resources:
```bash
kubectl get all -n default
kubectl get pvc
kubectl get secrets
```

### Check service status:
```bash
kubectl describe service <service-name>
kubectl logs deployment/<deployment-name>
```

## Security Considerations

1. **Network**: NGINX Ingress provides external access control
2. **Authentication**: JWT-based authentication across services
3. **Secrets Management**: Kubernetes Secrets store sensitive data (database passwords)

## Service Communications

- **Internal Communication**: Services use Kubernetes DNS (service-name.default.svc.cluster.local)
- **External Communication**: All traffic goes through NGINX Ingress

## Notes

- All services run in the `default` namespace in production
- Configuration management uses Kustomize (not Helm)
- Container images are sourced from Azure Container Registry
- Deployments include health checks for service reliability
