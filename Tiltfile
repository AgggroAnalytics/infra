# aggrov5 — Temporal, Postgres, RabbitMQ, MinIO, Keycloak, backend, workers
# Run: tilt up
# Port-forwards: API :8090, Temporal UI :8080, Keycloak :8180, Temporal gRPC :7233, MinIO, gateway :8081
#
# Images are built locally and loaded into k3d via `k3d image import`.
# No registry needed. imagePullPolicy: Never in all manifests.

K3D_CLUSTER = 'dev'

def k3d_build(ref, context, dockerfile='Dockerfile', **kwargs):
    custom_build(
        ref,
        'docker build -t $EXPECTED_REF -f %s/%s %s && k3d image import $EXPECTED_REF -c %s' % (context, dockerfile, context, K3D_CLUSTER),
        deps=[context],
        disable_push=True,
        skips_local_docker=True,
        **kwargs,
    )

k3d_build('aggro-backend', 'aggro-backend')
k3d_build('minio-gateway', 'minio-gateway')
k3d_build('aggro-field-worker', 'aggro-field-worker')

k8s_yaml([
    'infra/k8s/namespace.yaml',
    'infra/k8s/postgres.yaml',
    'infra/k8s/postgres-aggro.yaml',
    'infra/k8s/rabbitmq.yaml',
    'infra/k8s/temporal.yaml',
    'infra/k8s/temporal-ui.yaml',
    'infra/k8s/minio.yaml',
    'infra/k8s/minio-gateway.yaml',
    'infra/k8s/keycloak.yaml',
    'infra/k8s/aggro-backend.yaml',
    'infra/k8s/aggro-backend-temporal-worker.yaml',
    'infra/k8s/aggro-field-worker.yaml',
])

local_resource(
    'storage-provisioner-health',
    cmd='bash -c \'if kubectl auth can-i list persistentvolumeclaims --as=system:serviceaccount:kube-system:local-path-provisioner >/dev/null 2>&1; then echo "ok"; else echo "restarting"; kubectl rollout restart deployment/local-path-provisioner -n kube-system; kubectl rollout status deployment/local-path-provisioner -n kube-system --timeout=60s; fi\'',
    auto_init=True,
    trigger_mode=TRIGGER_MODE_AUTO,
    labels=['infra'],
)

k8s_resource('temporal-ui', port_forwards=['8080:8080'], labels=['temporal'])
k8s_resource('temporal', port_forwards=['7233:7233'], labels=['temporal'])
k8s_resource('postgres', labels=['data'], resource_deps=['storage-provisioner-health'])
k8s_resource('aggro-postgres', port_forwards=['5433:5432'], labels=['data'], resource_deps=['storage-provisioner-health'])
k8s_resource('rabbitmq', labels=['data'])
k8s_resource('minio', port_forwards=['9000:9000', '9001:9001'], labels=['minio'], resource_deps=['storage-provisioner-health'])
k8s_resource('minio-gateway', port_forwards=['8081:8080'], labels=['minio'])

local_resource(
    'minio-cors',
    cmd='kubectl delete job minio-cors -n aggrov5 --ignore-not-found=true && kubectl apply -f infra/k8s/minio-cors.yaml',
    deps=['infra/k8s/minio-cors.yaml'],
    resource_deps=['minio'],
    labels=['minio'],
)

k8s_resource('keycloak', port_forwards=['8180:8080'], labels=['auth'])

local_resource(
    'keycloak-realm-bootstrap',
    cmd='kubectl delete job keycloak-realm-bootstrap -n aggrov5 --ignore-not-found=true && kubectl apply -f infra/k8s/keycloak-realm-job.yaml && kubectl wait --for=condition=complete job/keycloak-realm-bootstrap -n aggrov5 --timeout=180s',
    deps=['infra/k8s/keycloak-realm-job.yaml'],
    resource_deps=['keycloak'],
    labels=['auth'],
)

k8s_resource(
    'aggro-backend',
    port_forwards=['8090:8080'],
    labels=['backend'],
    resource_deps=['aggro-postgres', 'rabbitmq', 'temporal', 'keycloak-realm-bootstrap'],
)
k8s_resource(
    'aggro-backend-temporal-worker',
    labels=['backend'],
    resource_deps=['aggro-postgres', 'temporal'],
)
k8s_resource(
    'aggro-field-worker',
    labels=['worker'],
    resource_deps=['temporal', 'minio', 'aggro-backend-temporal-worker'],
)
