# Docker registry (локалка + VPS)

В кластере поднимается **registry:2** (`infra/k8s/docker-registry.yaml`): PVC, Deployment, Service с **фиксированным `clusterIP`** (по умолчанию **10.43.253.50** под k3d — в хвосте /16, чтобы реже бить с авто-выданными IP).

## Почему не DNS `docker-registry.aggrov5.svc.cluster.local`

**Pull образа делает containerd на ноде**, не под. На ноде **нет** резолва `*.svc.cluster.local` → `no such host`.

**Решение:** в имени образа использовать **ClusterIP:порт** (например `10.43.253.50:5000/...`), который нода достигает через kube-proxy.

## TLS и pull/push

**containerd** на ноде по умолчанию ходит в registry по **HTTPS**. Если registry отвечает только по **HTTP**, будет ошибка `HTTP response to HTTPS client`.

В манифесте registry включён **HTTPS** с самоподписанным сертификатом (SAN: `clusterIP`, `127.0.0.1`, DNS сервиса). Для нод k3s нужен **`infra/k3d/k3s-registries.yaml`**: `https://` + `insecure_skip_verify: true`.

```bash
k3d cluster create dev --registry-config "$(pwd)/infra/k3d/k3s-registries.yaml"
```

Подробнее: **[infra/k3d/README.md](../k3d/README.md)**.

### `docker push` с хоста (Tilt → 127.0.0.1:30500)

У сертификата свой CA; Docker Engine обычно требует **insecure-registries** для такого endpoint:

`/etc/docker/daemon.json` (или Docker Desktop → Settings → Docker Engine):

```json
{
  "insecure-registries": ["127.0.0.1:30500"]
}
```

Перезапусти Docker. Без этого `docker push` на `https://127.0.0.1:30500/...` может падать на проверке сертификата.

## Локально (Tilt / k3d)

1. Кластер k3d **с** `--registry-config` (см. выше).
2. `tilt up`: push с хоста на `127.0.0.1:30500` через **port-forward** на сервис registry.
3. В YAML образы идут как **`10.43.253.50:5000/...`** (или `TILT_REGISTRY_CLUSTER_HOST`).

Переменные:

- `TILT_REGISTRY_PUSH_HOST` — куда `docker push` (по умолчанию `127.0.0.1:30500`).
- `TILT_REGISTRY_CLUSTER_HOST` — префикс образа для pull с ноды (по умолчанию `10.43.253.50:5000`).

Проверка:

```bash
kubectl get svc -n aggrov5 docker-registry
curl -sk https://127.0.0.1:30500/v2/ | head   # при работающем tilt port-forward
```

### kind / другой Service CIDR

У kind часто **10.96.0.0/12**. Задай другой свободный **`clusterIP`** в `docker-registry.yaml`, тот же хост в `TILT_REGISTRY_CLUSTER_HOST` и настрой insecure registry для kind (отдельно от k3d).

## Прод на VPS

1. Свой домен + **TLS** у registry (Ingress), в манифестах полный `image: registry.example.com/...`.
2. Фиксированный `clusterIP` из dev-манифеста можно убрать, если все pull идут через внешнее имя.
3. **Не** публиковать NodePort с голым HTTP registry наружу.

### Без своего in-cluster registry

**GHCR / GitLab / Docker Hub (private)** — полные имена образов и `imagePullSecrets`.

## Переменные

| Переменная | Назначение |
|------------|------------|
| `TILT_REGISTRY_PUSH_HOST` | Куда `docker push` с хоста (по умолчанию `127.0.0.1:30500`) |
| `TILT_REGISTRY_CLUSTER_HOST` | Адрес registry для имён образов в YAML (**должен резолвиться/достигаться с ноды**, по умолчанию `10.43.253.50:5000`) |
