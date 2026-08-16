# platform-infra

Kubernetes infrastructure and deployment manifests for running and managing multiple applications and their supporting services across development and other environments.

## Structure

```
platform-infra/
├── apps/
│   └── url-shortner/          # URL Shortener application manifests
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── configmap.yaml
│       └── secret.yaml
└── infrastructure/
    ├── mysql/                 # MySQL manifests (Deployment/StatefulSet, Service, PVC, ConfigMap, Secret)
    └── redis/                 # Redis manifests (Deployment, Service, PVC, ConfigMap, Secret)
```

The repo is split into two top-level concerns:
- **`apps/`** — application-specific manifests (one subfolder per app)
- **`infrastructure/`** — shared supporting services (databases, caches) that apps depend on

## Prerequisites

- A running Kubernetes cluster (Minikube, Kind, or Docker Desktop's built-in K8s)
- `kubectl` configured to point at that cluster
- Application Docker images already built and available to the cluster (see the [url-shortner repo](#) for build instructions)

## Deploying

### 1. Create the namespace

```bash
kubectl create namespace url-shortner
```

### 2. Deploy infrastructure (MySQL, Redis)

```bash
kubectl apply -f infrastructure/mysql/ -n url-shortner
kubectl apply -f infrastructure/redis/ -n url-shortner
```

Wait for both to be ready before deploying the app:
```bash
kubectl get pods -n url-shortner
```

### 3. Deploy the application

```bash
kubectl apply -f apps/url-shortner/ -n url-shortner
```

### 4. Verify

```bash
kubectl get all -n url-shortner
```

Once the app Pod is `Running` and `Ready`, access it via NodePort:
```
http://localhost:30080
```
(Adjust host/IP if not using Minikube/Docker Desktop directly — e.g. `minikube ip` for the actual node address.)

## Application: url-shortner

| Resource | File | Purpose |
|---|---|---|
| Deployment | [`deployment.yaml`](apps/url-shortner/deployment.yaml) | Runs the app container, 1 replica, with liveness/readiness probes against Spring Boot Actuator |
| Service | [`service.yaml`](apps/url-shortner/service.yaml) | Exposes the app via NodePort `30080` |
| ConfigMap | [`configmap.yaml`](apps/url-shortner/configmap.yaml) | Non-sensitive config — DB host/port/name, Redis host/port |
| Secret | [`secret.yaml`](apps/url-shortner/secret.yaml) | Sensitive config — DB username/password |

### Health checks

The app exposes dedicated liveness/readiness endpoints via Spring Boot Actuator:
- Liveness: `/actuator/health/liveness`
- Readiness: `/actuator/health/readiness`

These require `management.endpoint.health.probes.enabled=true` (and related properties) to be set in the application's own config — see the app repo.

## Infrastructure: MySQL & Redis

| Resource | File | Purpose |
|---|---|---|
| MySQL | [`infrastructure/mysql/`](infrastructure/mysql/) | Deployment/StatefulSet, Service, PVC, ConfigMap, Secret |
| Redis | [`infrastructure/redis/`](infrastructure/redis/) | Deployment, Service, PVC, ConfigMap, Secret |

> _Per-file breakdown pending — manifests are being finalized, including the choice between Deployment and StatefulSet for MySQL._

## Design Notes

- **Service-name-based discovery**: the app connects to `shortlink-mysql` / `shortlink-redis` (Service names), not Pod IPs — Kubernetes' internal DNS resolves these to the correct, currently-live Pod(s) even as Pods are recreated.
- **Secrets caveat**: Kubernetes `Secret` objects are base64-encoded, not encrypted, by default. This is acceptable for local/dev use in this repo; a real production setup would add etcd encryption at rest and/or an external secrets manager (Vault, AWS Secrets Manager, Sealed Secrets).
- **NodePort for local access**: used here for simplicity in a local/dev cluster. A production setup would front the Service with an Ingress (TLS termination, domain-based routing) instead.

## Known Gaps / Next Steps

- [ ] Finalize MySQL manifests — confirm StatefulSet vs Deployment, wire up PVC
- [ ] Finalize Redis manifests
- [ ] Add Ingress in front of the app Service
- [ ] Add HPA (Horizontal Pod Autoscaler) for the app Deployment
- [ ] Consider moving Secrets out of git entirely (external secrets manager or `.gitignore` + example templates)