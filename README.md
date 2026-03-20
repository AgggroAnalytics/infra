# aggrov5 infra (Kubernetes + Tilt)

## Stack (`tilt up`)

| Service | Role | Port-forward (Tilt) |
|---------|------|---------------------|
| **temporal** | Workflow engine | **7233** |
| **temporal-ui** | Web UI | **8080** |
| **keycloak** | OIDC; realm **aggro** (саморегистрация), client **aggro-frontend** | **8180** |
| **keycloak-realm-bootstrap** | Job (Tilt): создаёт realm без пользователей + SPA-клиент | — |
| **aggro-postgres** | PostGIS DB for aggro-backend | **5433→5432** |
| **postgres** | Legacy PVC (Temporal dev uses embedded DB) | — |
| **rabbitmq** | Tile/geo/ML job queues for API | — |
| **minio** | Object storage | **9000**, console **9001** |
| **minio-gateway** | HTTP gateway to MinIO buckets | **8081** |
| **aggro-backend** | HTTP API + JWT (Keycloak) + CORS для :5173 | **8090** |
| **aggro-backend-temporal-worker** | Activity `finalize_field_processing_db` | — |
| **aggro-field-worker** | `FieldProcessingWorkflow` | — |

### Keycloak

- Админ-консоль: http://localhost:8180 — **admin / admin**
- Realm **aggro**: на странице входа есть **Register** (саморегистрация, без подтверждения email в dev).
- Фронт: `VITE_KEYCLOAK_URL=http://localhost:8180`, realm `aggro`, client `aggro-frontend`.
- Job **`keycloak-realm-bootstrap`** при каждом `tilt up` пересоздаётся и идемпотентно создаёт realm и клиент (пользователей не создаёт).

## Prerequisites

- Kubernetes (kind, minikube, Docker Desktop, **k3d**)
- [Tilt](https://docs.tilt.dev/install.html)
- **Docker registry** в кластере: `infra/k8s/docker-registry.yaml`. В **Tilt** пуш идёт через **port-forward на :30500**, NodePort в Tilt не подключается. Подробно: **[infra/docs/REGISTRY.md](docs/REGISTRY.md)**.

GEE: [aggro-field-worker/secrets/README.md](../aggro-field-worker/secrets/README.md).

## Run

```bash
cd /path/to/aggrov5
tilt up
```

- API: http://localhost:8090  
- OpenAPI: http://localhost:8090/openapi.yaml  
- Keycloak: http://localhost:8180  
- Temporal UI: http://localhost:8080  

```bash
TEMPORAL_ADDRESS=localhost:7233 uv run --project aggro-field-worker python run_field_workflow.py
```

**PMTiles:** воркер шлёт URL с `minio-gateway`; с хоста используй порт **8081**.

## kind

```bash
kind create cluster
tilt up
```

## Manifests (`infra/k8s/`)

| File | Contents |
|------|----------|
| `keycloak.yaml` | Keycloak `start-dev`, hostname `http://localhost:8180` (iss в JWT) |
| `keycloak-realm-job.yaml` | ConfigMap + Job: realm **aggro** + client **aggro-frontend** |
| `aggro-backend.yaml` | `KEYCLOAK_ISSUER`, JWKS, `CORS_ALLOW_ORIGINS` для Vite |
| … | (остальные как раньше) |
