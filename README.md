# aggrov5 infra (Kubernetes + Tilt)

- **Temporal** – workflow engine (port 7233)
- **Temporal UI** – http://localhost:8080 (after port-forward)
- **PostgreSQL** – used by Temporal
- **MinIO** – object storage for observed/predicted PMTiles (9000, 9001). Tilt runs the `minio-cors` job on every `tilt up` so shared object URLs work in the browser.
- **aggro-field-worker** – Temporal worker (GEE, ML, PMTiles via [felt/tippecanoe](https://github.com/felt/tippecanoe))

## Prerequisites

- Kubernetes cluster (e.g. minikube, kind, Docker Desktop k8s)
- [Tilt](https://docs.tilt.dev/install.html)

GEE credentials are **not** in k8s; they are handled by **aggro-field-worker** via SOPS. See [aggro-field-worker/secrets/README.md](../aggro-field-worker/secrets/README.md).

## Run with Tilt

From the **aggrov5** repo root:

```bash
tilt up
```

Tilt will:

- Build `aggro-field-worker` and load it into your cluster (use a cluster that uses the local Docker daemon, e.g. kind or minikube with docker driver, so `imagePullPolicy: Never` works).
- Deploy namespace, postgres, temporal, temporal-ui, minio, aggro-field-worker.
- Port-forward Temporal UI to 8080 and Temporal to 7233.

To run the workflow from your machine, ensure port 7233 is forwarded, then:

```bash
# From aggrov5 root
TEMPORAL_ADDRESS=localhost:7233 uv run --project aggro-field-worker python run_field_workflow.py
```

If the worker runs inside the cluster, use `kubectl port-forward -n aggrov5 svc/temporal 7233:7233` if Tilt didn’t set it up.

## Local Kubernetes (kind)

So that the worker image is available in the cluster:

```bash
kind create cluster
tilt up
# Tilt will build and load the image into kind
```

## Manifest reference

| File | Contents |
|------|----------|
| `k8s/namespace.yaml` | Namespace `aggrov5` |
| `k8s/postgres.yaml` | PostgreSQL for Temporal |
| `k8s/temporal.yaml` | Temporal server (auto-setup) |
| `k8s/temporal-ui.yaml` | Temporal Web UI |
| `k8s/minio.yaml` | MinIO server |
| `k8s/aggro-field-worker.yaml` | Field processing worker (GEE credentials via SOPS in worker, not k8s) |
