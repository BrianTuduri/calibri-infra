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
kubectl create namespace sonrisas
kubectl create secret generic pg-superuser \
  -n sonrisas \
  --from-literal=username=postgres \
  --from-literal=password="$(openssl rand -base64 24)"

# 2b. Redis auth (ns data). Redis exige password.
kubectl create namespace data
kubectl create secret generic redis-auth \
  -n data \
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
  --docker-password='<TU_GITHUB_PAT_CON_SCOPE_READ_PACKAGES>'

# 2e. App secrets (ns sonrisas). DATABASE_URL y REDIS_URL se completan en el paso 4
#     (dependen de CNPG y del password de Redis). El resto, valores reales ya.
kubectl create secret generic app-secrets \
  -n sonrisas \
  --from-literal=NEXTAUTH_SECRET="$(openssl rand -base64 32)" \
  --from-literal=NEXTAUTH_URL="https://api.dentalink.healthatom.com/api/v1" \
  --from-literal=DENTALINK_API_KEY='CnMZ68OFJ7svAnlvaIZttxj76mD76SxCAAt4ufYx.DlImAHa3ixXLLE86qhMFBvoptFnE2icxxNfWOm9e' \
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

# 5 Garage Bootstrap — Estado y pasos pendientes

## Ya ejecutado (por Sisyphus)

Todos los comandos necesitan `export KUBECONFIG=/home/mgorospe/.kube/k3s-hetzner.kube`

```bash
# 1. Obtener node ID
kubectl -n data exec garage-0 -- /garage -c /etc/garage/garage.toml node id
# Resultado: ecdc39f0065801e7f3321e4bd113e4965da908804ba67255a7e7a785ed283486

# 2. Asignar nodo al layout (con 10G de capacidad — con 1 fallaba)
kubectl -n data exec garage-0 -- /garage -c /etc/garage/garage.toml layout assign \
  ecdc39f0065801e7f3321e4bd113e4965da908804ba67255a7e7a785ed283486 \
  -z default -c 10G -t garage-0

# 3. Aplicar layout
kubectl -n data exec garage-0 -- /garage -c /etc/garage/garage.toml layout apply --version 1
# Resultado: 256 particiones, 10 GB usables, replication factor 1

# 4. Crear buckets
kubectl -n data exec garage-0 -- /garage -c /etc/garage/garage.toml bucket create sonrisas-reports
kubectl -n data exec garage-0 -- /garage -c /etc/garage/garage.toml bucket create sonrisas-pg
# Ambos creados OK
```

## Pendiente — ejecutar manualmente

### Paso 5: Crear la key S3

```bash
kubectl -n data exec garage-0 -- /garage -c /etc/garage/garage.toml key create sonrisas-app
```

Esto imprime algo como:

```
Key name: sonrisas-app
Key ID: GKxxxxxxxxxxxxxxxxxxxx
Secret key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**Guardar esos dos valores** — los necesitás abajo.

### Paso 6: Dar permisos a la key sobre los buckets

```bash
kubectl -n data exec garage-0 -- /garage -c /etc/garage/garage.toml bucket allow sonrisas-reports \
  --read --write --key sonrisas-app

kubectl -n data exec garage-0 -- /garage -c /etc/garage/garage.toml bucket allow sonrisas-pg \
  --read --write --key sonrisas-app
```

### Paso 7: Guardar las keys en Kubernetes

Reemplazar `<KEY_ID>` y `<SECRET>` con los valores del paso 5:

```bash
KEY_ID="<KEY_ID>"
SECRET="<SECRET>"

# Patchear app-secrets con las keys S3
kubectl -n sonrisas patch secret app-secrets --type merge \
  -p "{\"stringData\":{\"S3_ACCESS_KEY_ID\":\"$KEY_ID\",\"S3_SECRET_ACCESS_KEY\":\"$SECRET\"}}"

# Crear secret para backups CNPG
kubectl create secret generic garage-s3-creds -n sonrisas \
  --from-literal=ACCESS_KEY_ID="$KEY_ID" \
  --from-literal=SECRET_ACCESS_KEY="$SECRET"
```

### Paso 8: Restart de los deployments (cuando existan)

```bash
kubectl -n sonrisas rollout restart deploy
```

## Nota importante

El script original usaba `garage` sin `-c`, pero la imagen distroless busca
`/etc/garage.toml` por defecto y el ConfigMap lo monta en `/etc/garage/garage.toml`.
**Todos los comandos manuales necesitan `-c /etc/garage/garage.toml`.**

## Otro problema pendiente: migrate job

El job `migrate` falla con `ImagePullBackOff` porque la imagen
`ghcr.io/manuelgorospe/sonrisas-worker:1.0.0` no existe en GHCR.
Hasta que se publique esa imagen, el pipeline de la app está bloqueado

---

## 6. (Opcional) Backups de Postgres → Garage

Cuando Garage esté operativo: descomentá el bloque `backup:` en
`components/postgres/base/cluster.yaml`, commiteá y ArgoCD lo aplica. El Secret
`garage-s3-creds` ya quedó creado en el paso 5.

---

## 7. GlitchTip (error tracking)

GlitchTip corre en el ns `glitchtip` con Postgres como cola (sin Valkey) y su base
`glitchtip` vive en el Cluster `sonrisas-pg`. Solo se accede por Tailscale:
`https://glitchtip.tailc13b86.ts.net`.

Agregar el rol y la base a CNPG **no reinicia Postgres** (no están entre los disparadores
de rolling update de CNPG).

> **Atención:** el kubeconfig de prod ya se pisó una vez con un login a OKD. Cada bloque
> arranca con el chequeo de contexto: si no imprime `btuduri@cygnus-k3s`, pará.

### 7a. Secrets (antes del push)

El password se genera una vez y se reusa en los dos Secrets. Es hex, así que no
necesita url-encode en el `DATABASE_URL`. Corré el bloque entero en la misma terminal.

```bash
export KUBECONFIG=~/.kube/k3s-hetzner-prod.kube
kubectl config current-context   # debe ser btuduri@cygnus-k3s

GT_PASS=$(openssl rand -hex 24)

kubectl create secret generic glitchtip-pg-role \
  -n sonrisas \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=glitchtip \
  --from-literal=password="$GT_PASS"
kubectl label secret glitchtip-pg-role -n sonrisas cnpg.io/reload=true

kubectl create namespace glitchtip
kubectl create secret generic glitchtip-secrets \
  -n glitchtip \
  --from-literal=SECRET_KEY="$(openssl rand -hex 32)" \
  --from-literal=DATABASE_URL="postgres://glitchtip:${GT_PASS}@sonrisas-pg-rw.sonrisas.svc:5432/glitchtip"

unset GT_PASS
```

### 7b. Push y sync

Pusheá en dos tandas. Si va todo junto, el Job de migraciones puede correr antes de que
CNPG cree la base, falla, y Argo no reintenta la misma revisión.

Primero solo el commit de `components/postgres`. Esperá a que la base exista:

```bash
export KUBECONFIG=~/.kube/k3s-hetzner-prod.kube
kubectl config current-context   # debe ser btuduri@cygnus-k3s

kubectl -n sonrisas get database glitchtip   # APPLIED tiene que estar en true
kubectl -n sonrisas exec sonrisas-pg-1 -c postgres -- psql -U postgres -c '\l' | rg glitchtip
```

Después pusheá el resto. Argo sincroniza `glitchtip`: primero el Job, después el Deployment.

```bash
kubectl -n glitchtip get job,pod
kubectl -n glitchtip logs job/glitchtip-migrate
```

Esperado: Job `Complete`, pod `1/1 Running` y la UI respondiendo en
`https://glitchtip.tailc13b86.ts.net`. Si el log del pod muestra `too many connections for
role "glitchtip"`, subí `connectionLimit` en `cluster.yaml`.

### 7c. Alta inicial

```bash
export KUBECONFIG=~/.kube/k3s-hetzner-prod.kube
kubectl config current-context   # debe ser btuduri@cygnus-k3s

kubectl -n glitchtip exec -it deploy/glitchtip -- ./manage.py createsuperuser
```

En la UI creá la organización `calabri` y los proyectos `sonrisas-web` y
`sonrisas-workers`. Anotá la public key y el project id de cada uno: están en el DSN
que muestra la UI (`https://<public_key>@glitchtip.tailc13b86.ts.net/<project_id>`).

### 7d. DSN para la app

> **Atención:** la UI muestra el DSN con el host de Tailscale. Los pods no resuelven
> `*.ts.net` y el SDK falla en silencio. Armá el DSN con el host interno.

```bash
export KUBECONFIG=~/.kube/k3s-hetzner-prod.kube
kubectl config current-context   # debe ser btuduri@cygnus-k3s

WEB_DSN="http://<PUBLIC_KEY_WEB>@glitchtip-web.glitchtip.svc.cluster.local:8000/<PROJECT_ID_WEB>"
WORKERS_DSN="http://<PUBLIC_KEY_WORKERS>@glitchtip-web.glitchtip.svc.cluster.local:8000/<PROJECT_ID_WORKERS>"

kubectl -n sonrisas patch secret app-secrets --type merge \
  -p "{\"stringData\":{\"SENTRY_DSN\":\"$WEB_DSN\",\"SENTRY_DSN_WORKERS\":\"$WORKERS_DSN\"}}"

kubectl -n sonrisas rollout restart deploy/sonrisas-web deploy/sonrisas-sync-worker \
  deploy/sonrisas-alert-worker deploy/sonrisas-report-worker
```

### NetworkPolicy

Si entra la NetworkPolicy default-deny de "Seguridad pendiente", tiene que permitir
`sonrisas` → `glitchtip` (puerto 8000: DSN de los workers y tunnel de la web) y
`glitchtip` → `sonrisas-pg` (5432). Sin lo primero los eventos se pierden en silencio;
sin lo segundo GlitchTip no arranca.

---

## 8. Landing (calabri.net)

La web pública corre en el ns `landing`: un Deployment Node (Astro standalone, puerto 4321)
detrás de Traefik, con su propio Certificate para `calabri.net` y `www.calabri.net`. `www`
redirige al apex con el Middleware `www-to-apex`. El código vive en `calabri1/calabri-web`;
acá solo el deploy.

> **Atención:** no pushees `apps/landing.yaml` a `main` antes de que exista la imagen que
> fija el overlay (`newTag`). Argo sincroniza solo y los pods quedan en `ImagePullBackOff`.

### 8a. Primera imagen

En `calabri-web`, taggeá la versión que fija el overlay:

```bash
git tag v0.1.1
git push origin v0.1.1
```

El workflow `Release` publica `ghcr.io/calabri1/calabri-web:v0.1.1`. Verificá en
GitHub → Packages que exista y que el usuario del PAT de `ghcr-pull` pueda leerlo.

> **Atención:** no deployes `v0.1.0`. Usa Astro 5, y un header `Host` con puerto inválido
> tira el proceso Node (GHSA-qh8j-hqjv-7m4x). Desde `v0.1.1` el sitio corre en Astro 7.

### 8b. Secrets (antes del push)

```bash
export KUBECONFIG=~/.kube/k3s-hetzner-prod.kube
kubectl config current-context   # debe ser btuduri@cygnus-k3s

kubectl create namespace landing

kubectl get secret ghcr-pull -n sonrisas -o json \
  | jq 'del(.metadata.namespace,.metadata.uid,.metadata.resourceVersion,.metadata.creationTimestamp,.metadata.managedFields)' \
  | kubectl apply -n landing -f -
```

El Secret del formulario es `landing-secrets`, con las variables que lee `/api/contact`. El
Deployment lo marca `optional`: la página levanta sin él, así que un Secret faltante o mal
cargado recién aparece cuando alguien manda el formulario. Probalo después del deploy (8d).

El envío sale por Amazon SES con el usuario IAM `ses-smtp-web`. Sin `CONTACT_FROM`, cada envío
sale con remitente vacío y SES lo rechaza.

```bash
kubectl create secret generic landing-secrets \
  -n landing \
  --from-literal=SMTP_HOST=email-smtp.eu-central-1.amazonaws.com \
  --from-literal=SMTP_PORT=587 \
  --from-literal=SMTP_USER='<usuario SMTP de ses-smtp-web>' \
  --from-literal=SMTP_PASS='<password SMTP de ses-smtp-web>' \
  --from-literal=CONTACT_FROM=web@notify.calabri.net \
  --from-literal=CONTACT_TO=contacto@calabri.net \
  --from-literal=TURNSTILE_SECRET_KEY='<secret key de Turnstile>'
```

SES está en `eu-central-1`; credenciales SMTP de otra región fallan con `535`. Detalle y
troubleshooting en `calabri-web/docs/formulario-contacto.md`.

En Cloudflare → Turnstile, el widget tiene que tener `calabri.net` en su lista de hostnames.
Si no, el widget muestra el error `110200` y nadie puede mandar el formulario.

La site key de Turnstile no va en este Secret: el formulario se prerenderiza y la clave se
fija al construir la imagen. Cargala como variable del repo `calabri-web` (Settings →
Secrets and variables → Actions → Variables) con el nombre `PUBLIC_TURNSTILE_SITE_KEY`.
Sin ella el workflow `Release` falla a propósito, para no publicar la clave de prueba.

### 8c. DNS en Cloudflare

Creá `calabri.net` (apex) y `www` con el mismo destino que `sonrisas.calabri.net`, ambos
con proxy activado. Antes, fijate que el apex no tenga ya otro registro: no lo verifiqué.

### 8d. Push y verificación

Pusheá `apps/landing.yaml` y `components/landing/`. Argo emite primero el Certificate
(wave 1) y después levanta el resto.

```bash
export KUBECONFIG=~/.kube/k3s-hetzner-prod.kube
kubectl config current-context   # debe ser btuduri@cygnus-k3s

kubectl -n landing get certificate,pod,ingress   # Certificate READY True, pods 1/1
curl -sI https://calabri.net | head -1           # HTTP/2 200
curl -sI https://www.calabri.net | rg -i '^(HTTP|location)'   # 301 a https://calabri.net/
```

Mandá el formulario desde `https://calabri.net` y confirmá que llega a `contacto@calabri.net`.

Si el Certificate queda en `False`, mirá `kubectl -n landing describe certificaterequest`:
el DNS-01 necesita el mismo token de Cloudflare que usa el wildcard.

> **Atención:** el límite de envíos por IP confía en el header `cf-connecting-ip`. Si el
> nodo acepta tráfico 443 que no venga de Cloudflare, cualquiera lo falsifica y se saltea
> el límite. Cerrá el puerto en el firewall de Hetzner a los rangos de Cloudflare.

### 8e. Nuevas versiones

Taggeá `vX.Y.Z` en `calabri-web`, esperá el workflow `Release` y subí `newTag` en
`components/landing/overlays/prod/kustomization.yaml`.

---

## Dominio del Ingress

Editá `components/sonrisas/base/ingress.yaml` y reemplazá `sonrisas.calabri.net` por el
dominio real (ej: `sonrisas.tuduri.com.uy`). Commiteá → ArgoCD lo aplica.
Requiere cert-manager + ClusterIssuer `letsencrypt-prod` en el cluster.

> El **código fuente** de la app sigue en `sonrisasuy`; acá vive sólo el deploy
> (cómo se levanta en el cluster). Las imágenes se buildean en el CI de `sonrisasuy`
> y se publican en `ghcr.io/calabri1/sonrisas-{web,worker}`. La versión de prod
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


---
