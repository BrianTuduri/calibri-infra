# Pasos manuales — bootstrap del cluster k3s

Este repo es la **infra** (ArgoCD + Postgres/CNPG + Redis + Garage). La **app**
(web/workers) vive en `sonrisasuy` y ArgoCD la trae sola en la última wave.

Acá sólo está lo que GitOps **no puede hacer solo**: instalar ArgoCD y crear los
Secrets. Todo lo demás es commit → ArgoCD sincroniza. **No apliques nada de
`components/` a mano.**

Orden de arranque (todo lo necesario primero, la app al final):

```
1. Instalar ArgoCD
2. Crear los Secrets (una vez)
3. Aplicar el app-of-apps  ──►  ArgoCD sincroniza por waves:
                                  wave 0  cnpg-operator
                                  wave 1  postgres · redis · garage
                                  wave 2  sonrisas (la app, sola)
4. Completar app-secrets (DATABASE_URL / REDIS_URL) post-sync
5. Bootstrap de Garage + claves S3
6. (opcional) Backups de Postgres
```

Las waves son reales: están como `argocd.argoproj.io/sync-wave` en cada Application.
Cada wave espera a que la anterior esté Healthy antes de seguir.

---

## 0. Pre-vuelo

```bash
# El StorageClass debe permitir expansión (Postgres pide 50Gi).
kubectl get storageclass local-path -o jsonpath='{.allowVolumeExpansion}{"\n"}'
# "true" → ok.  "false" → pará y avisá: hay que reconfigurar local-path antes.

# Disco libre en el nodo (la DB + Garage viven en local-path, mismo disco).
kubectl get nodes -o wide
```

---

## 1. Instalar ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update

helm install argocd argo/argo-cd \
  -n argocd --create-namespace \
  --version 9.5.x \
  -f bootstrap/argocd-values.yaml

kubectl -n argocd wait --for=condition=Ready pod --all --timeout=120s
```

---

## 2. Crear los Secrets (una sola vez, antes del primer sync)

Sin estos Secrets los pods quedan en `CreateContainerConfigError`. Creálos **todos
juntos acá** — no se repiten en ningún otro paso.

```bash
# 2a. Superusuario de Postgres (ns sonrisas). CNPG lo usa para inicializar.
kubectl create secret generic pg-superuser \
  -n sonrisas --create-namespace \
  --from-literal=username=postgres \
  --from-literal=password="$(openssl rand -base64 24)"

# 2b. Redis auth (ns data). Redis exige password.
kubectl create secret generic redis-auth \
  -n data --create-namespace \
  --from-literal=password="$(openssl rand -base64 32)"

# 2c. Garage RPC secret (ns data). Exactamente 64 hex chars.
kubectl create secret generic garage-rpc-secret \
  -n data \
  --from-literal=RPC_SECRET="$(openssl rand -hex 32)"

# 2d. Pull secret para ghcr.io (ns sonrisas). PAT con scope read:packages.
kubectl create secret docker-registry ghcr-pull \
  -n sonrisas \
  --docker-server=ghcr.io \
  --docker-username=ManuelGorospe \
  --docker-password='<GITHUB_PAT>'

# 2e. App secrets (ns sonrisas). DATABASE_URL y REDIS_URL se completan en el paso 4
#     (dependen de CNPG y del password de Redis). El resto, valores reales ya.
kubectl create secret generic app-secrets \
  -n sonrisas \
  --from-literal=NEXTAUTH_SECRET="$(openssl rand -base64 32)" \
  --from-literal=NEXTAUTH_URL="https://SONRISAS_DOMAIN" \
  --from-literal=DENTALINK_API_KEY='<KEY>' \
  --from-literal=S3_BUCKET_NAME="sonrisas-reports"
  # DATABASE_URL, REDIS_URL, S3_ACCESS_KEY_ID, S3_SECRET_ACCESS_KEY → pasos 4 y 5.
```

> La app/migrate van a CrashLoop hasta el paso 4. Es esperable: ArgoCD y Kubernetes
> reintentan solos en cuanto el Secret esté completo.

---

## 3. Aplicar el app-of-apps

```bash
kubectl apply -f bootstrap/root-app.yaml
```

A partir de acá ArgoCD descubre `apps/` y sincroniza por waves automáticamente.

---

## 4. Completar `app-secrets` (post-sync de postgres y redis)

Cuando `postgres` (wave 1) esté Healthy, CNPG genera el Secret `sonrisas-pg-app` con
la URI lista. Tomala de ahí en vez de escribirla a mano:

```bash
# DATABASE_URL desde la URI que generó CNPG (usuario/host/password correctos)
DBURL=$(kubectl -n sonrisas get secret sonrisas-pg-app -o jsonpath='{.data.uri}' | base64 -d)

# REDIS_URL con el password de Redis, url-encoded (base64 trae / + =)
REDIS_PASS=$(kubectl -n data get secret redis-auth -o jsonpath='{.data.password}' | base64 -d)
REDIS_PASS_ENC=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1], safe=''))" "$REDIS_PASS")

kubectl -n sonrisas patch secret app-secrets --type merge -p "$(cat <<EOF
{"stringData":{
  "DATABASE_URL":"$DBURL",
  "REDIS_URL":"redis://:${REDIS_PASS_ENC}@redis.data.svc.cluster.local:6379"
}}
EOF
)"

# Reiniciar lo que ya hubiera arrancado con secrets incompletos
kubectl -n sonrisas rollout restart deploy
```

---

## 5. Bootstrap de Garage + claves S3 (cuando `garage-0` esté Running)

```bash
NODE_ID=$(kubectl -n data exec garage-0 -- garage node id 2>/dev/null | head -1)

kubectl -n data exec garage-0 -- garage layout assign "$NODE_ID" -z default -c 1 -t garage-0
kubectl -n data exec garage-0 -- garage layout apply --version 1

kubectl -n data exec garage-0 -- garage bucket create sonrisas-reports
kubectl -n data exec garage-0 -- garage bucket create sonrisas-pg   # backups CNPG
kubectl -n data exec garage-0 -- garage key create sonrisas-app
kubectl -n data exec garage-0 -- garage bucket allow sonrisas-reports --read --write --key sonrisas-app
kubectl -n data exec garage-0 -- garage bucket allow sonrisas-pg     --read --write --key sonrisas-app
```

Garage imprime `Key ID` y `Secret key`. Cargalas en la app y guardalas para backups:

```bash
KEY_ID="<KEY_ID>"; SECRET="<SECRET>"

# S3 keys de la app
kubectl -n sonrisas patch secret app-secrets --type merge \
  -p "{\"stringData\":{\"S3_ACCESS_KEY_ID\":\"$KEY_ID\",\"S3_SECRET_ACCESS_KEY\":\"$SECRET\"}}"

# Mismas keys para los backups de CNPG (paso 6)
kubectl create secret generic garage-s3-creds -n sonrisas \
  --from-literal=ACCESS_KEY_ID="$KEY_ID" \
  --from-literal=SECRET_ACCESS_KEY="$SECRET"

kubectl -n sonrisas rollout restart deploy
```

El endpoint S3 (`http://garage.data.svc.cluster.local:3900`) y la región ya están en
`components/sonrisas/base/app-config.yaml`.

---

## 6. (Opcional) Backups de Postgres → Garage

Cuando Garage esté operativo: descomentá el bloque `backup:` en
`components/postgres/base/cluster.yaml`, commiteá y ArgoCD lo aplica. El Secret
`garage-s3-creds` ya quedó creado en el paso 5.

---

## Dominio del Ingress

Editá `components/sonrisas/base/ingress.yaml` y reemplazá `SONRISAS_DOMAIN` por el
dominio real (ej: `sonrisas.tuduri.com.uy`). Commiteá → ArgoCD lo aplica.
Requiere cert-manager + ClusterIssuer `letsencrypt-prod` en el cluster.

> El **código fuente** de la app sigue en `sonrisasuy`; acá vive sólo el deploy
> (cómo se levanta en el cluster). Las imágenes se buildean en el CI de `sonrisasuy`
> y se publican en `ghcr.io/manuelgorospe/sonrisas-{web,worker}`. La versión de prod
> se fija con `newTag` en `components/sonrisas/overlays/prod/kustomization.yaml`
> (hoy `1.0.0`). Para subir de versión: publicar la imagen con ese tag y bumpear acá.

---

## Post-backfill — bajar a valores normales

Los manifiestos de la app vienen tuneados para el backfill inicial de ~3 años. Cuando
`sync_config.ultimo_sync` de todos los recursos esté al día, commiteá estos cambios:

| Archivo | Cambio |
|---|---|
| `components/sonrisas/base/app-config.yaml` | `SYNC_LOOKBACK_DAYS: "14"` |
| `components/sonrisas/base/workers.yaml` | sync-worker: `PG_POOL_MAX=8`, `UV_THREADPOOL_SIZE=8`, heap `512`, requests `200m/384Mi`, limits `1500m/768Mi` |
| `components/postgres/base/cluster.yaml` | `shared_buffers=512MB`, `maintenance_work_mem=128MB`, `max_connections=60`, requests `500m/768Mi`, limits `1500m/2Gi` |

El storage de Postgres (50Gi) **no** se baja — Kubernetes no permite shrink de PVCs.

---

## Seguridad pendiente (post-arranque)

- **Backups off-cluster (crítico):** Garage y Postgres están en `local-path`, mismo
  disco. Si muere el disco perdés DB y backups juntos. Apuntá el backup de CNPG (paso 6)
  a un bucket externo (B2/R2/Wasabi) o espejá con `rclone`.
- **Imagen de Garage:** hoy es `dianlettre/garage:v1.0.1` (Docker Hub de un tercero).
  Mejor migrar a la oficial `dxflrs/garage` pineada por digest.
- **NetworkPolicy egress:** default-deny + allowlist (Dentalink, kube-dns, GHCR).
  Probar en staging primero.
- **Pinear imágenes de infra por digest:** CNPG Postgres (`ghcr.io/cloudnative-pg/postgresql:16`)
  y el operador CNPG.
