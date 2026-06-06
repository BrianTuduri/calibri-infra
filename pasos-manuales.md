# Pasos manuales — k3s GitOps bootstrap

Estos pasos se ejecutan **una sola vez** antes del primer sync de ArgoCD.
Todo lo posterior es GitOps puro (commit → ArgoCD sincroniza).

---

## 1. Instalar ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update

helm install argocd argo/argo-cd \
  -n argocd --create-namespace \
  --version 9.5.x \
  -f gitops/bootstrap/argocd-values.yaml
```

Esperar a que los pods estén Ready:

```bash
kubectl -n argocd wait --for=condition=Ready pod --all --timeout=120s
```

---

## 2. Aplicar App-of-Apps

```bash
kubectl apply -f gitops/bootstrap/root-app.yaml
```

A partir de aquí ArgoCD descubre todo `gitops/apps/` y sincroniza automáticamente.
**No aplicar nada más de `gitops/` a mano.**

---

## 3. Secrets requeridos antes del primer sync útil

Crear estos secrets **antes** de que ArgoCD intente sincronizar los componentes,
o el sync fallará con `CreateContainerConfigError`.

### 3a. Superusuario de Postgres (namespace `sonrisas`)

```bash
kubectl create secret generic pg-superuser \
  -n sonrisas --create-namespace \
  --from-literal=username=postgres \
  --from-literal=password=$(openssl rand -base64 24)
```

> CNPG usa este Secret para inicializar el cluster. Guardá la contraseña generada.
> Después del primer sync, CNPG crea automáticamente el Secret `sonrisas-pg-app`
> con la URI de conexión lista para usar como `DATABASE_URL`.

### 3b. Redis auth (namespace `data`)

```bash
kubectl create secret generic redis-auth \
  -n data --create-namespace \
  --from-literal=password=$(openssl rand -base64 24)
```

### 3c. Garage RPC secret (namespace `data`)

```bash
kubectl create secret generic garage-rpc-secret \
  -n data \
  --from-literal=RPC_SECRET=$(openssl rand -hex 32)
```

> Debe ser exactamente 64 caracteres hex. El comando de arriba genera 32 bytes → 64 hex chars.

### 3d. Image pull secret para ghcr.io (namespace `sonrisas`)

```bash
kubectl create secret docker-registry ghcr-pull \
  -n sonrisas \
  --docker-server=ghcr.io \
  --docker-username=ManuelGorospe \
  --docker-password=<GITHUB_PAT>
```

> El PAT necesita scope `read:packages`.

### 3e. App secrets (namespace `sonrisas`)

Crear con los valores reales del entorno:

```bash
kubectl create secret generic app-secrets \
  -n sonrisas \
  --from-literal=DATABASE_URL="postgresql://sonrisas:PASSWORD@sonrisas-pg-rw.sonrisas.svc.cluster.local:5432/sonrisas" \
  --from-literal=REDIS_URL="redis://:PASSWORD@redis.data.svc.cluster.local:6379" \
  --from-literal=NEXTAUTH_SECRET="$(openssl rand -base64 32)" \
  --from-literal=NEXTAUTH_URL="https://SONRISAS_DOMAIN" \
  --from-literal=DENTALINK_API_KEY="<KEY>" \
  --from-literal=S3_ACCESS_KEY_ID="PLACEHOLDER" \
  --from-literal=S3_SECRET_ACCESS_KEY="PLACEHOLDER" \
  --from-literal=S3_BUCKET_NAME="sonrisas-reports"
```

> `DATABASE_URL` usa `sonrisas-pg-rw` (el Service que crea CNPG para escritura).
> `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` se actualizan en el paso 5 (post-Garage).

---

## 4. Actualizar dominio en el Ingress

Editar `gitops/components/sonrisas/base/ingress.yaml` y reemplazar `SONRISAS_DOMAIN`
con el dominio real (ej: `sonrisas.tuduri.com.uy`). Commitear y pushear — ArgoCD lo aplica.

Requiere cert-manager + ClusterIssuer `letsencrypt-prod` en el cluster.

---

## 5. Bootstrap de Garage (post-primer-sync)

Una vez que el pod `garage-0` esté Running en namespace `data`:

```bash
# Ver el node ID
NODE_ID=$(kubectl -n data exec garage-0 -- garage node id 2>/dev/null | head -1)

# Asignar layout (zona default, 1 copia, tag = nombre del pod)
kubectl -n data exec garage-0 -- garage layout assign \
  "$NODE_ID" -z default -c 1 -t garage-0

# Confirmar layout
kubectl -n data exec garage-0 -- garage layout apply --version 1

# Crear buckets
kubectl -n data exec garage-0 -- garage bucket create sonrisas-reports
kubectl -n data exec garage-0 -- garage bucket create sonrisas-pg   # para backups CNPG

# Crear key de acceso para la app
kubectl -n data exec garage-0 -- garage key create sonrisas-app

# Dar permisos
kubectl -n data exec garage-0 -- garage bucket allow sonrisas-reports \
  --read --write --key sonrisas-app
kubectl -n data exec garage-0 -- garage bucket allow sonrisas-pg \
  --read --write --key sonrisas-app
```

Garage imprime el `Key ID` y `Secret key`. Guardarlos y actualizar el Secret de la app:

```bash
kubectl -n sonrisas patch secret app-secrets \
  --type='json' \
  -p='[
    {"op":"replace","path":"/data/S3_ACCESS_KEY_ID","value":"'$(echo -n KEY_ID | base64)'"},
    {"op":"replace","path":"/data/S3_SECRET_ACCESS_KEY","value":"'$(echo -n SECRET | base64)'"}
  ]'
```

---

## 6. Habilitar backups de Postgres (post-Garage)

Descomentar el bloque `backup:` en `gitops/components/postgres/base/cluster.yaml`.
Crear Secret `garage-s3-creds` en namespace `sonrisas`:

```bash
kubectl create secret generic garage-s3-creds \
  -n sonrisas \
  --from-literal=ACCESS_KEY_ID="<KEY_ID>" \
  --from-literal=SECRET_ACCESS_KEY="<SECRET>"
```

Commitear el cluster.yaml con el bloque backup descomentado → ArgoCD aplica.

---

## Resumen de dependencias de sync

```
cnpg-operator  (wave 0, Helm)
     ↓
postgres       (wave 1 — espera CRDs de CNPG)
redis
garage
     ↓
sonrisas       (wave 2 — migrate job primero, luego deployments)
```

ArgoCD maneja las waves automáticamente vía las anotaciones `argocd.argoproj.io/sync-wave`.

---

# Backfill productivo k3s — sync de 3 años de Dentalink

> Estos pasos aplican al cluster **k3s** (namespace `sonrisas`), gestionado por el repo `gitops/`.
> Los cambios de manifiesto ya están en el repo; quedan SOLO las acciones manuales que GitOps no puede hacer.
>
> Ejecutar **en este orden**.

## 0. Pre-vuelo (revisar antes de pushear)

```bash
# Verificar que el StorageClass local-path permite expansión (para el PVC de Postgres).
kubectl get storageclass local-path -o jsonpath='{.allowVolumeExpansion}{"\n"}'
# "true"  → expansión in-place (camino A, sección 3).
# "false" → recreate dance (camino B, sección 3) — STOP y leé antes de pushear.

# Ver cuánto disco libre hay en el nodo
kubectl get nodes -o wide
# y en el nodo: df -h /var/lib/rancher/k3s/storage
```

## 1. Crear `redis-auth` ANTES de pushear

Redis ahora exige password. Sin este secret el pod queda en `CreateContainerConfigError`.

```bash
kubectl -n data create secret generic redis-auth \
  --from-literal=password="$(openssl rand -base64 32)"

kubectl -n data annotate secret redis-auth \
  argocd.argoproj.io/sync-options=Prune=false --overwrite
```

## 2. Actualizar `REDIS_URL` en `app-secrets`

El secret de la app tiene la URL de Redis sin password. Ahora hay que agregarla.

```bash
REDIS_PASS=$(kubectl -n data get secret redis-auth -o jsonpath='{.data.password}' | base64 -d)

# URL-encode del password (base64 puede traer / + =)
REDIS_PASS_ENC=$(python3 -c \
  "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1], safe=''))" \
  "$REDIS_PASS")

kubectl -n sonrisas patch secret app-secrets --type merge \
  -p "{\"stringData\":{\"REDIS_URL\":\"redis://:${REDIS_PASS_ENC}@redis.data.svc.cluster.local:6379\"}}"
```

## 3. Expand del PVC de Postgres (10 Gi → 50 Gi)

El CNPG Cluster ya declara `storage: 50Gi` en el manifiesto. CNPG maneja la expansión automáticamente si el StorageClass lo permite.

### Camino A — `allowVolumeExpansion: true` (lo normal en k3s con local-path)

Aplicá el manifiesto y CNPG solicita el resize al PVC. Verificar:

```bash
kubectl -n sonrisas get pvc -l cnpg.io/cluster=sonrisas-pg
# Esperar que CAPACITY muestre 50Gi. Puede tardar unos minutos.
```

Si el PVC no crece luego del sync, forzar la solicitud:

```bash
kubectl -n sonrisas patch pvc \
  $(kubectl -n sonrisas get pvc -l cnpg.io/cluster=sonrisas-pg -o name | head -1) \
  -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

### Camino B — StorageClass sin expansión

CNPG no puede redimensionar el PVC → el sync va a fallar con error en el Cluster.

```bash
# 1. Pausar ArgoCD sobre la app postgres (que no resincronice)
argocd app set postgres --sync-policy none

# 2. Dump antes de tocar nada
kubectl -n sonrisas exec -it \
  $(kubectl -n sonrisas get pod -l cnpg.io/cluster=sonrisas-pg,role=primary -o name) -- \
  pg_dump -U sonrisas -d sonrisas -Fc > /tmp/sonrisas-pre-resize-$(date -u +%Y%m%dT%H%M%SZ).dump

# 3. STOP → consultá con Brian antes de continuar.
#    La única salida es cambiar el StorageClass del nodo o importar los datos en un PVC nuevo.
```

## 4. Actualizar el dominio en el Ingress

El manifiesto `infra/deploy/k3s/prod/ingress.yaml` tiene el placeholder `SONRISAS_DOMAIN`.
Reemplazarlo con el dominio real antes de pushear:

```bash
# Editar el archivo y reemplazar SONRISAS_DOMAIN con el dominio real (ej: sonrisas.tuduri.com.uy)
# El edit puede hacerse en el repo antes del commit.
```

## 5. Pushear los cambios

```bash
git add \
  infra/deploy/k3s/ \
  gitops/apps/sonrisas.yaml \
  gitops/components/postgres/base/cluster.yaml \
  gitops/components/redis/base/redis.yaml \
  gitops/pasos-manuales.md
git commit -m "feat(deploy): k3s prod manifests + tune for 3-year Dentalink backfill"
git push
```

ArgoCD va a detectar el cambio en la Application `sonrisas` (ahora apunta a `infra/deploy/k3s/prod`).
Forzar un refresh si no sincroniza solo:

```bash
argocd app get sonrisas --refresh hard
argocd app sync sonrisas
```

## 6. Verificar que el backfill arrancó

```bash
# Sync-worker levantando con SYNC_LOOKBACK_DAYS=1100
kubectl -n sonrisas logs deploy/sonrisas-sync-worker | grep -i "lookback\|since\|1100"

# Cola de BullMQ creciendo (con password)
REDIS_PASS=$(kubectl -n data get secret redis-auth -o jsonpath='{.data.password}' | base64 -d)
kubectl -n data exec statefulset/redis -- \
  redis-cli -a "$REDIS_PASS" --no-auth-warning LLEN bull:sync:wait

# Progreso por recurso en la DB
kubectl -n sonrisas exec -it \
  $(kubectl -n sonrisas get pod -l cnpg.io/cluster=sonrisas-pg,role=primary -o name) -- \
  psql -U sonrisas -d sonrisas -c \
  "SELECT recurso, ultimo_sync FROM sync_config ORDER BY ultimo_sync NULLS FIRST;"

# Estado general
kubectl -n sonrisas get pods
kubectl -n sonrisas top pod
```

## 7. Mientras corre el backfill — qué monitorear

| Métrica | Comando | Alarma |
|---|---|---|
| Memoria Postgres | `kubectl -n sonrisas top pod -l cnpg.io/cluster=sonrisas-pg` | > 90% del limit |
| Disco Postgres | `kubectl -n sonrisas exec <pod-pg> -- df -h /var/lib/postgresql/data` | > 80% |
| Cola Redis | `redis-cli -a $REDIS_PASS LLEN bull:sync:wait` | crece > 100k sin bajar |
| 429 Dentalink | `kubectl -n sonrisas logs deploy/sonrisas-sync-worker \| grep "429\|rate"` | repetido |
| OOMKills | `kubectl -n sonrisas get events \| grep OOM` | cualquiera |

Si pega 429 sostenido, es rate-limit de Dentalink. Revisar si el sync-worker tiene backoff configurable en `packages/sync/`.

## 8. POST-backfill — bajar recursos a valores normales

Cuando `sync_config.ultimo_sync` de todos los recursos esté al día (< 1 día atrás):

```bash
# No hay cronjob que suspender en k3s (el backup es manejado por CNPG + Garage cuando esté activo).
```

Commitear estos cambios de rollback en el repo:

| Archivo | Cambio |
|---|---|
| `infra/deploy/k3s/prod/app-config.yaml` | `SYNC_LOOKBACK_DAYS: "14"` (no 7 — Dentalink modifica records hasta ~10 días atrás) |
| `infra/deploy/k3s/prod/workers.yaml` | sync-worker: `PG_POOL_MAX=8`, `UV_THREADPOOL_SIZE=8`, `NODE_OPTIONS=--max-old-space-size=512`, requests `200m/384Mi`, limits `1500m/768Mi` |
| `gitops/components/postgres/base/cluster.yaml` | `shared_buffers=512MB`, `maintenance_work_mem=128MB`, `max_connections=60`, requests `500m/768Mi`, limits `1500m/2Gi` |
| `gitops/components/redis/base/redis.yaml` | resources requests `100m/256Mi`, limits `500m/2Gi` (el `requirepass` NO se revierte) |

**El storage de Postgres (50Gi) NO se baja** — Kubernetes no permite shrinking de PVCs.

## 9. Seguridad pendiente (atender después del backfill)

- **Backups off-cluster (CRÍTICO)**: Garage y Postgres viven en `local-path` en el mismo nodo. Si se muere el disco, perdés DB y backups simultáneamente. Habilitar el bloque `backup:` de CNPG en `gitops/components/postgres/base/cluster.yaml` apuntando a un bucket externo (B2/R2/Wasabi) una vez que Garage esté operativo, o usar `rclone` para mirror.
- **NetworkPolicy egress**: hoy los pods pueden salir a cualquier IP. Agregar default-deny egress + allowlist (Dentalink API, kube-dns, GHCR) — probarlo primero en staging.
- **Pinear imágenes de infraestructura**: `redis:7.4-alpine` ya está versionado. Faltan los digests de la imagen CNPG Postgres (`ghcr.io/cloudnative-pg/postgresql:16`) y del operador CNPG.

