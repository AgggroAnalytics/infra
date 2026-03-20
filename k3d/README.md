# k3d и in-cluster registry

Образы подтягивает **containerd на ноде**. Он **не** резолвит `*.svc.cluster.local`. Registry в кластере слушает **HTTPS** (самоподписанный TLS); в `k3s-registries.yaml` указаны `https://` и `insecure_skip_verify`.

## Создание кластера

Из корня репозитория:

```bash
k3d cluster create dev \
  --registry-config "$(pwd)/infra/k3d/k3s-registries.yaml"
```

Если кластер уже есть — пересоздай с тем же флагом или добавь эквивалентный конфиг на ноды вручную (сложнее).

## Смена Service CIDR / конфликт IP

Если `10.43.253.50` занят или другой CIDR:

1. Поменяй `clusterIP` в `infra/k8s/docker-registry.yaml`.
2. Обнови `infra/k3d/k3s-registries.yaml` (те же host:port).
3. Выставь `export TILT_REGISTRY_CLUSTER_HOST=<ip>:5000` или поправь дефолт в `Tiltfile`.

После смены `clusterIP` у существующего Service может понадобиться: `kubectl delete svc docker-registry -n aggrov5` и снова `kubectl apply`.
