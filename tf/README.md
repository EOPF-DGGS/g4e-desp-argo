# Grid4Earth DESP — JupyterHub + Argo Workflows on OVHcloud

OpenTofu deployment of JupyterHub, Dask Gateway and Argo Workflows on the GRID4EARTH OVHcloud project (GRA9).

## Stack

- JupyterHub 4.3.2 (Zero-to-JupyterHub)
- Dask Gateway 2025.4.0
- Argo Workflows 0.46.2
- NGINX Ingress Controller
- cert-manager v1.14.4 + Let's Encrypt
- nginx-s3-gateway (S3 public proxy)
- OVHcloud MKS (Managed Kubernetes Service, GRA9)
- State stored in OVH S3 bucket (`g4e-desp-state`)

## Prerequisites

- OpenTofu >= 1.6
- `kubectl`
- `helm`
- OVH API credentials (`OVH_APPLICATION_KEY`, `OVH_APPLICATION_SECRET`, `OVH_CONSUMER_KEY`)
- S3 credentials for the GRID4EARTH project (see Step 1)

---

## Step 1 — Retrieve S3 credentials

S3 credentials are managed from the OVHcloud Manager:

1. Log in to https://manager.eu.ovhcloud.com with the GRID4EARTH account
2. Navigate to **Public Cloud → GRID4EARTH → Object Storage → Users**
3. Select an existing S3 user (or create one) and click **View credentials**

The `access_key` and `secret_key` values are needed for `backend.tfvars` and `secrets/terraform.tfvars`.

---

## Step 2 — Configure OVH API credentials

Create an API token at https://www.ovh.com/auth/api/createToken with GET/POST/PUT/DELETE rights on `/*`.

Save the credentials in `secrets/ovh-creds.sh` (encrypted with git-crypt):

```bash
export OVH_ENDPOINT="ovh-eu"
export OVH_APPLICATION_KEY="..."
export OVH_APPLICATION_SECRET="..."
export OVH_CONSUMER_KEY="..."
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="gra"
export AWS_REGION="gra"
```

Source before any Tofu operation:

```bash
git-crypt unlock ~/git-crypt-g4e.key
source secrets/ovh-creds.sh
```

The git-crypt key is stored at `~/git-crypt-g4e.key`. Keep it in a safe place (password manager, USB key) — without it the secrets file cannot be decrypted.

---

## Step 3 — Create the S3 state bucket

The state bucket must exist before running `tofu init`:

```bash
AWS_ACCESS_KEY_ID="<s3_access_key>" \
AWS_SECRET_ACCESS_KEY="<s3_secret_key>" \
aws s3 mb s3://g4e-desp-state \
  --endpoint-url https://s3.gra.io.cloud.ovh.net \
  --region gra
```

---

## Step 4 — Configure secrets

Create `backend.tfvars` (not committed):

```hcl
access_key = "<s3_access_key>"
secret_key = "<s3_secret_key>"
```

Create `secrets/terraform.tfvars` (not committed):

```hcl
harbor_robot_username = "<robot_username>"
harbor_robot_token    = "<robot_token>"
s3_access_key         = "<s3_access_key>"
s3_secret_key         = "<s3_secret_key>"
s3proxy_access_key    = "<s3proxy_access_key>"
s3proxy_secret_key    = "<s3proxy_secret_key>"
```

The `s3proxy_access_key` / `s3proxy_secret_key` are the credentials for the `grid4earth` bucket
served publicly via `https://data.grid4earth.eu`.

The CDSE (Copernicus Data Space Ecosystem) login used by the orbit pipeline is not a Tofu
variable: its secret is created with `kubectl`, see [CDSE credentials](#cdse-credentials).

---

## Step 5 — Initialize Tofu

```bash
cd tf
git-crypt unlock ~/git-crypt-g4e.key
source secrets/ovh-creds.sh

tofu init -reconfigure
```

If prompted for backend credentials, make sure `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
are set (they come from `secrets/ovh-creds.sh`).

---

## Step 6 — Deploy

The deployment must be done in four passes due to a Tofu limitation: the `kubernetes_manifest`
resource (cert-manager ClusterIssuer) cannot be planned before the cluster exists.

**Pass 1 — Create the cluster and node pool:**

```bash
tofu apply \
  -var-file=secrets/terraform.tfvars \
  -target=ovh_cloud_project_kube.cluster \
  -target=ovh_cloud_project_kube_nodepool.cpu_pool
```

This takes approximately 7–10 minutes.

**Pass 2 — Deploy all Helm releases and Kubernetes resources:**

```bash
tofu apply \
  -var-file=secrets/terraform.tfvars \
  -target=helm_release.cert_manager \
  -target=helm_release.ingress_nginx \
  -target=kubernetes_namespace.jupyterhub \
  -target=kubernetes_namespace.argo \
  -target=kubernetes_namespace.s3proxy \
  -target=kubernetes_secret.harbor_pull_secret \
  -target=kubernetes_secret.harbor_pull_secret_argo \
  -target=kubernetes_secret.argo_s3_credentials \
  -target=kubernetes_secret.s3_credentials_jupyterhub \
  -target=kubernetes_secret.s3proxy_credentials \
  -target=helm_release.jupyterhub \
  -target=helm_release.dask_gateway \
  -target=helm_release.argo_workflows \
  -target=kubernetes_network_policy.singleuser_dask \
  -target=kubernetes_network_policy.singleuser_to_argo
```

This takes approximately 10–15 minutes (first run pulls the singleuser image, ~5.8 GiB).

**Pass 3 — Deploy the cert-manager ClusterIssuer:**

```bash
tofu apply \
  -var-file=secrets/terraform.tfvars \
  -target=kubernetes_manifest.cluster_issuer
```

**Pass 4 — Deploy the STAC stack and S3 proxy:**

```bash
kubectl apply -f grid4earth-stac-stack.yaml
kubectl apply -f grid4earth-s3proxy.yaml
```

Monitor TLS certificate issuance:

```bash
kubectl get certificate -A --watch
```

---

## Step 7 — Configure DNS

The domain `grid4earth.eu` is registered under Tina's OVHcloud account, which is separate from
the GRID4EARTH infrastructure account (Fred's account, `pf81809-ovh`). The DNS zone is managed
by OVH nameservers (`ns109.ovh.net` / `dns109.ovh.net`).

A single wildcard A record covers all subdomains (already configured — no action needed):

```
*.grid4earth.eu  →  <LoadBalancer external IP>  TTL 300
```

After Pass 2, retrieve the LoadBalancer external IP:

```bash
kubectl get svc -n jupyterhub ingress-nginx-controller
```

TLS certificates are issued automatically by cert-manager (Let's Encrypt) within a few minutes
of the DNS record propagating.

---

## Step 8 — Deploy the STAC stack

After the cluster and DNS are in place, deploy the STAC stack:

```bash
kubectl apply -f grid4earth-stac-stack.yaml
```

---

## Step 9 — Retrieve kubeconfig

The kubeconfig is available from the OVHcloud Manager:

**Public Cloud → GRID4EARTH → Managed Kubernetes Service → g4e-desp-cluster → kubeconfig → Download**

```bash
export KUBECONFIG=~/.kube/config-desp
kubectl get pods -n jupyterhub
kubectl get pods -n argo
kubectl get pods -n s3proxy
```

---

## Architecture

```
OVHcloud project: GRID4EARTH (24b43ff90f3044c8923063b0fbb53f26)
│
├── MKS Cluster (GRA9) — g4e-desp-cluster
│   └── Node pool CPU: b3-64, autoscale 1–5
│       label: hub.jupyter.org/node-purpose=user, node-role=cpu
│
├── Namespace: jupyterhub
│   ├── JupyterHub 4.3.2     — https://jupyterhub.grid4earth.eu
│   │   ├── Profile: Standard CPU (✅ operational) — up to 32 GB RAM
│   │   ├── Profile: Sentinel-2 MSI (✅ operational) — up to 64 GB RAM
│   │   │   allowed users: pablo-richard, capetienne, cgueguen, j34ni, annefou
│   │   ├── Profile: Sentinel-3 SYNERGY / s3syn (✅ operational) — up to 32 GB RAM
│   │   │   allowed users: j34ni, annefou, tik65536, tinaok
│   │   │   image: y74y55mn.gra7.container-registry.ovh.net/healpix-private/s3syn:latest
│   │   │   base: quay.io/jupyter/minimal-notebook:2024-05-27
│   │   │   s3syn version: 1.0.6
│   │   └── Profile: GPU (⚠️  disabled — see GPU section below)
│   ├── Dask Gateway 2025.4.0
│   ├── NGINX Ingress (class: nginx-jupyterhub)
│   └── cert-manager — Let's Encrypt TLS
│
├── Namespace: argo
│   └── Argo Workflows 0.46.2 — https://argo.grid4earth.eu
│       └── Artifacts → S3 bucket (TBD — see Argo artifacts section)
│
├── Namespace: stac
│   ├── stac-fastapi-geoparquet — https://stac-api.grid4earth.eu
│   └── stac-browser            — https://stac-browser.grid4earth.eu
│       (custom build: ghcr.io/j34ni/stac-browser:gridlook)
│
├── Namespace: gridlook
│   └── gridlook                — https://gridlook.grid4earth.eu
│
├── Namespace: s3proxy
│   └── nginx-s3-gateway        — https://data.grid4earth.eu
│       Proxies s3://grid4earth/public/ without exposing credentials.
│       Managed by: grid4earth-s3proxy.yaml (Deployment/Service/Ingress)
│                   main.tf (Namespace + Secret)
│       Usage: https://data.grid4earth.eu/<path>
│              maps to s3://grid4earth/public/<path>
│
└── S3 buckets (GRA)
    ├── g4e-desp-state          (Tofu state)
    ├── grid4earth              (public data via data.grid4earth.eu)
    └── <TBD>                   (Argo Workflows artifacts)
```

---

## JupyterHub profiles

### s3syn profile

The s3syn profile provides the Sentinel-3 SYNERGY Level-2 processor environment.

**Image build** — built from source on a VM from the
[synergy-processor](https://gitlab.eopf.copernicus.eu/S3/SYN/synergy-processor) repository,
then pushed to Harbor:

```bash
# Clone
git clone https://<user>:<token>@gitlab.eopf.copernicus.eu/S3/SYN/synergy-processor.git

# Build (token needed for private EOPF package registries)
docker build \
  --build-arg GITLAB_TOKEN="<token>" \
  -t s3syn-jupyter:latest \
  -f Dockerfile_s3syn \
  .

# Push to Harbor
docker tag s3syn-jupyter:latest \
  y74y55mn.gra7.container-registry.ovh.net/healpix-private/s3syn:latest
docker push y74y55mn.gra7.container-registry.ovh.net/healpix-private/s3syn:latest
```

**Private package registries** used during build (full list from the
[s3syn installation manual](https://s3.pages.eopf.copernicus.eu/SYN/synergy-processor/main/sim.html)):

| Project ID | Package |
|---|---|
| 519 | s3syn |
| 118 | s3olci |
| 92 | asgard-legacy |
| 171 | asgard-legacy-drivers |
| 102, 113, 14, 78, 94, 52, 67 | other EOPF dependencies |

**Quick install check** in a notebook:

```python
import s3syn
print(s3syn.__version__)  # should print 1.0.6

from s3syn.sy1.computing.sy1_processor import Sy1Processor
from s3syn.sy2aod.computing.aod_processing_unit import AODProcessing
print("OK:", Sy1Processor, AODProcessing)
```

**Allowed users:** `j34ni`, `annefou`, `tik65536`, `tinaok`

---

## S3 public proxy

The S3 proxy at `https://data.grid4earth.eu` provides public read-only access to the
`grid4earth` S3 bucket without exposing credentials. It is backed by
[nginxinc/nginx-s3-gateway](https://github.com/nginxinc/nginx-s3-gateway).

Files in `s3://grid4earth/public/` are accessible at:

```
https://data.grid4earth.eu/<path>
```

Example:

```bash
curl https://data.grid4earth.eu/tmp/test.zarr/zarr.json
```

CORS is fully open (`Access-Control-Allow-Origin: *`) with support for `Range` requests,
which is required for Zarr and cloud-optimised formats.

CORS headers are injected via an nginx `configuration-snippet` on the Ingress (not via the
`enable-cors` annotations, which only apply to OPTIONS preflight responses). This requires
two settings in the ingress-nginx Helm release, already configured in `main.tf`:

```hcl
controller.allowSnippetAnnotations    = true
controller.config.annotations-risk-level = Critical
```

> **Note on ingress-nginx >= 1.12:** since chart version 4.12, `allowSnippetAnnotations: true`
> alone is not sufficient — `annotations-risk-level: Critical` is also required, otherwise the
> admission webhook rejects `configuration-snippet` annotations.

---

## Harbor — Private Registry

The JupyterHub singleuser image is hosted on the OVHcloud Harbor registry:

```
y74y55mn.gra7.container-registry.ovh.net/healpix-private/g4e-jupyterhub-private:latest
```

The robot account used for pulling is a project-level robot in the `healpix-private` project.
Credentials are stored in `secrets/terraform.tfvars` and injected automatically as a Kubernetes
`imagePullSecret` (`harbor-pull-secret`) in both the `jupyterhub` and `argo` namespaces.

When using `docker login` from the CLI, use single quotes around the username to prevent shell
expansion of the `$` character:

```bash
echo 'TOKEN' | docker login y74y55mn.gra7.container-registry.ovh.net \
  -u 'robot$healpix-private+<robot-name>' \
  --password-stdin
```

---

## GPU support

GRA9 currently offers **Quadro RTX 5000 (16 GB VRAM)** nodes. The GPU profile in JupyterHub
is present in `values.yaml` but marked as unavailable pending a decision on whether this GPU
meets project requirements for HEALPix regridding workloads.

When GPU nodes are provisioned:

1. Verify nodes appear in the cluster:
   ```bash
   kubectl get nodes -l node-role=gpu
   ```

2. Add the GPU node pool in `main.tf` with the correct `flavor_name`.

3. Deploy the NVIDIA device plugin:
   ```bash
   helm upgrade --install nvidia-device-plugin \
     https://nvidia.github.io/k8s-device-plugin/stable/nvidia-device-plugin.tgz \
     -n nvidia-device-plugin --create-namespace \
     -f nvidia-plugin-values.yaml
   ```

4. Update the GPU profile `display_name` in `values.yaml` to remove the "not available yet" warning.

---

## Argo Workflows artifacts

Argo is configured to store artifacts and logs in an S3 bucket. The bucket name is defined in
`argo-values.yaml`. Once the bucket name is confirmed, create it:

```bash
AWS_ACCESS_KEY_ID="<s3_access_key>" \
AWS_SECRET_ACCESS_KEY="<s3_secret_key>" \
aws s3 mb s3://<bucket-name> \
  --endpoint-url https://s3.gra.io.cloud.ovh.net \
  --region gra
```

Then update `argo-values.yaml` accordingly and redeploy:

```bash
tofu apply -var-file=secrets/terraform.tfvars -target=helm_release.argo_workflows
```

---

## Orbit pipeline

The orbit pipeline selects all the Sentinel products of one satellite orbit (same
`sat:absolute_orbit`) over France in the [CDSE STAC](https://stac.dataspace.copernicus.eu/v1/),
mirrors them as EOPF Zarr (UTM) in `s3://grid4earth/public/eopf-mirror/<collection>/`, converts
the whole orbit strip into a single HEALPix Zarr in `s3://grid4earth/public/converted/<collection>/`
and refreshes the public STAC index with [stac-scraper](https://github.com/GRID4EARTH/stac-scraper).
It is made of four WorkflowTemplates in `workflows/`:

| WorkflowTemplate | File | Role |
|---|---|---|
| `cdse-orbit-to-eopf-mirror` | `workflows/cdse-orbit-to-eopf-mirror.yaml` | CDSE search of the orbit, then one pod per product (at most 4 at a time): copy from the EOPF Sample Service when it has every product of the orbit, else CDSE SAFE download and conversion to EOPF Zarr, into `eopf-mirror/` (complete stores are skipped) |
| `healpix-convert-multistage` | `workflows/healpix-convert-multistage.yaml` | conversion of the orbit strip to one HEALPix Zarr in `converted/` (staging cache, prepare, chunk tasks, finalize) |
| `stac-scraper-update` | `workflows/stac-scraper-update.yaml` | re-scrapes `eopf-mirror` and `converted`, uploads the parquet files then `collections.json` (public-read) |
| `orbit-to-healpix-pipeline` | `workflows/orbit-to-healpix-pipeline.yaml` | checks the arguments and prerequisites, then chains the three templates above with `templateRef` (the STAC step only when `update_stac=true`) |

The pipeline uses three images, each a workflow parameter:

| Parameter | Default | Used for |
|---|---|---|
| `eopf_image` | `s2msi:fefecebf` | SAFE to EOPF Zarr (eopf) |
| `healpix_image` | `g4e-jupyterhub-private:2026-10-01` | CDSE search (pystac-client) |
| `orbit_image` | `g4e-jupyterhub-private:2026-10-10-orbit` | argument check, HEALPix conversion and STAC index (healpix-convert, legacy-converters, stac-scraper) |

`orbit_image` is built from `containers/Dockerfile_private_orbit` (see `containers/.build`) and
must be pushed before the first run. The first step of the pipeline (`check-arguments`) runs on
it: when the image is missing its pod stays in `ImagePullBackOff` and the run fails after 15
minutes (`activeDeadlineSeconds`), before anything is downloaded. Push the image and submit
again. With `-p orbit_image=<...>:2026-10-01 -p update_stac=false` the pipeline runs without it,
but the conversion then falls back to the converter bundled in the old `legacy_converters` (the
Sentinel-2 `conditions/geometry` and `conditions/meteorology` groups stay empty) and there is no
STAC update. Every pod has a deadline, so a pod that cannot start fails instead of waiting
forever; `argo stop -n argo <workflow>` stops a run by hand.

### CDSE credentials

Downloads from CDSE need an account. The workflows read it from the `argo-cdse-credentials`
secret (keys `username` and `password`), mounted read-only at `/etc/cdse`; it is never passed as a
workflow parameter. It is not managed by Tofu (a `tofu apply` would otherwise need the password
on every machine and could overwrite the secret): create it once with `kubectl`, prefixing the
command with a space or clearing the shell history so that the password is not kept there:

```bash
 kubectl -n argo create secret generic argo-cdse-credentials \
  --from-literal=username='<cdse_username>' \
  --from-literal=password='<cdse_password>'
```

`check-arguments` stops the run when this secret or `argo-s3-credentials` is missing.

Recommendations:

- Use an account dedicated to the pipeline rather than a personal one: every workflow submitted
  to the `argo` namespace can mount this secret.
- A CDSE account allows 4 concurrent connections. The mirror step runs at most 4 pods at a time
  for that reason; do not download with the same account elsewhere (for example
  `legacy-datasets` `download_orbit.py`) while a pipeline runs. HTTP 429 / 503 answers are waited
  out (`Retry-After`) and the access token is reused for a few minutes.

### Artifacts

`healpix-convert-multistage` (staging cache and task list) and `stac-scraper-update` (the
`stac-index` output) store their artifacts at an explicit location,
`s3://g4e-desp-argo-artifacts/<template>/<workflow uid>/`, with the `argo-s3-credentials` secret.
They do not rely on the default artifact repository: the `artifactRepositoryRef` block of
`argo-values.yaml` nests its key under `data:`, so the `artifact-repositories` ConfigMap rendered
by the chart has no `default-v1-s3` key and the controller has no usable default repository
(check with `kubectl -n argo get cm artifact-repositories -o yaml`).

### Deploying and running

From the repository root (the pipeline calls the other three with `templateRef`, so they must
all be deployed):

```bash
kubectl apply -n argo \
  -f tf/workflows/cdse-orbit-to-eopf-mirror.yaml \
  -f tf/workflows/healpix-convert-multistage.yaml \
  -f tf/workflows/stac-scraper-update.yaml \
  -f tf/workflows/orbit-to-healpix-pipeline.yaml
```

Example: Sentinel-2C orbit 4025 (relative orbit 51, 13 June 2025) over France, identified by one of
its products:

```bash
argo submit -n argo --watch \
  --from workflowtemplate/orbit-to-healpix-pipeline \
  -p reference_item=S2C_MSIL2A_20250613T104641_N0511_R051_T31UDQ_20250613T134507
```

The 65 products of the orbit (58 tiles; at a datatake boundary a tile can have two products, both
are kept) are mirrored in `s3://grid4earth/public/eopf-mirror/sentinel-2-l2a/` and the HEALPix
result is written to
`s3://grid4earth/public/converted/sentinel-2-l2a/S2C_MSIL2A_20250613T104641_R051_O4025_FRANCE.zarr`
(`{platform}_{type}_{first sensing start}_R{relative orbit}_O{absolute orbit}_{region}`).

Main parameters of `orbit-to-healpix-pipeline` (the others are described in the template):

| Parameter | Default | Meaning |
|---|---|---|
| `collection` | `sentinel-2-l2a` | CDSE collection (`sentinel-2-l1c`, `sentinel-2-l2a`, `sentinel-3-olci-1-efr-ntc`, `sentinel-3-sl-1-rbt-ntc`, ...) |
| `reference_item` | | one CDSE product of the orbit; otherwise give `datetime` (and `platform`, `absolute_orbit`, `relative_orbit`), not both |
| `bbox` | France | search area, `minx,miny,maxx,maxy` or `france` (empty = whole orbit) |
| `region_name` | | last part of the output name; empty = `FRANCE` for the France bbox, else `REGION` |
| `check_sample_service` | `true` | copy the products from the EOPF Sample Service when it has every product of the orbit (stores from both sources cannot be merged into one HEALPix dataset) |
| `settings_name` | `sentinel-2-l2a` | `legacy_converters` settings, named after the G4E collection: it **must match** `collection` (empty = derived from it); the run stops at the argument check otherwise |
| `groups` | all | JSON list of groups to convert, e.g. `'["measurements/reflectance/r60m"]'` |
| `overwrite` | `false` | replace an existing HEALPix output |
| `update_stac`, `stac_dry_run` | `true`, `false` | refresh the STAC index after the conversion, or only build it |
| `stac_allow_removals` | `false` | publish `collections.json` even when it drops collections of the published one |

For Sentinel-3, set `settings_name` to the G4E collection (or its `-psf` variant), e.g. for OLCI
EFR:

```bash
argo submit -n argo --watch \
  --from workflowtemplate/orbit-to-healpix-pipeline \
  -p collection=sentinel-3-olci-1-efr-ntc \
  -p settings_name=sentinel-3-olci-l1-efr \
  -p reference_item=S3B_OL_1_EFR____20250613T111915_20250613T112215_20250614T123815_0179_107_308_2160_ESA_O_NT_004
```

which writes
`s3://grid4earth/public/converted/sentinel-3-olci-l1-efr/S3B_OL_1_EFR_20250613T111615_R308_O37155_FRANCE.zarr`.

A full Sentinel-2 orbit over France is a large job: about 52 GB of SAFE downloads for 65
products, then tens of thousands of output chunks per 10 m group. For a first test, keep both the
mirror and the HEALPix output in `tmp/` (never indexed), use a small area (here 2 products around
Paris) and one group, without touching the STAC index:

```bash
argo submit -n argo --watch \
  --from workflowtemplate/orbit-to-healpix-pipeline \
  -p reference_item=S2C_MSIL2A_20250613T104641_N0511_R051_T31UDQ_20250613T134507 \
  -p bbox=2.2,48.7,2.5,48.9 -p region_name=PARIS \
  -p mirror_prefix=s3://grid4earth/public/tmp/eopf-mirror \
  -p converted_prefix=s3://grid4earth/public/tmp/converted \
  -p groups='["measurements/reflectance/r60m"]' \
  -p update_stac=false
```

Behaviour worth knowing:

- Re-running on the same orbit skips the products already mirrored (a store is complete when its
  root `zarr.json` has `stac_discovery`; an incomplete one left by a failed attempt is deleted and
  written again), but the conversion refuses an existing output unless `-p overwrite=true`. To
  resume a run whose conversion tasks failed, use `argo retry -n argo <workflow>` (only the failed
  tasks run again).
- The HEALPix dataset gets its root `stac_discovery`, which makes stac-scraper index it, only in
  the last conversion step (`finalize`), once every chunk is written; until then the item is kept
  as `stac_discovery_pending`. With `overwrite=true` the existing dataset is deleted at the start
  of the conversion and is not indexed again before `finalize`.
- Groups whose inputs cannot be concatenated are left out with a warning in the `stage-cache` log:
  for a Sentinel-2 orbit, `conditions/geometry`, whose `detector` dimension differs between full
  and partial tiles.
- `healpix-convert-multistage` works around three issues of healpix-convert 3aa72e0 until they
  are fixed upstream (see the comment at the top of the template): input STAC items with
  different `stac_extensions` are merged as a union, a chunk without any input point is left
  empty instead of failing, and the resamplers get float64 coordinates (the float32 ones of the
  `no_chunk` groups make them fail).
- Objects written under `s3://grid4earth/public/` are public-read, like the rest of that area.
- `stac-scraper-update` stops before scraping when a dataset directory has no root `zarr.json`
  (an interrupted upload by another tool: delete it), and does not publish a `collections.json`
  that drops published collections unless `allow_removals=true` (`stac_allow_removals=true` in the
  pipeline). The first update with stac-scraper 62b5c01 drops `converted-sentinel-3-synergy`,
  whose parquet file no longer exists: check it with `stac_dry_run=true`, then allow it once.
  stac-scraper has no collection template for `sentinel-3-slstr-l2-frp` and
  `sentinel-3-slstr-l2-lst`: their items are scraped but these collections are not listed in
  `collections.json`.

The STAC index can also be refreshed on its own. With `dry_run=true` nothing is uploaded and the
generated index is kept as the `stac-index` output artifact:

```bash
argo submit -n argo --watch \
  --from workflowtemplate/stac-scraper-update \
  -p dry_run=true
```

---

## STAC catalog

The following subdomains are deployed via `grid4earth-stac-stack.yaml`:

- stac-fastapi-geoparquet → https://stac-api.grid4earth.eu
- stac-browser → https://stac-browser.grid4earth.eu
- gridlook → https://gridlook.grid4earth.eu

---

## GitHub OAuth

The JupyterHub GitHub OAuth callback URL is:

```
https://jupyterhub.grid4earth.eu/hub/oauth_callback
```

This must match the callback URL configured in the GitHub OAuth App settings
(github.com → Settings → Developer settings → OAuth Apps).
