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
│   └── Argo Workflows 0.46.2 — in-cluster only (argo-workflows-server.argo.svc:2746)
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

The Argo server has no public ingress any more (the `argo.grid4earth.eu` ingress was removed);
PR #19 sets `server.ingress.enabled: false` in `argo-values.yaml` so that an apply does not create
it again. Until #19 is merged, do not apply `argo-values.yaml` from `main`.

---

## Orbit pipeline

The orbit pipeline selects the Sentinel products of one satellite orbit (same
`sat:absolute_orbit`) in the [CDSE STAC](https://stac.dataspace.copernicus.eu/v1/), cut as a
latitude band (by default **48.0..50.0 N**, around Paris; the preset `france` =
41.3..51.2 N stays available) over the full swath width, mirrors them as EOPF Zarr (UTM) in
`s3://grid4earth/public/eopf-mirror/<collection>/`, converts the strip into a single HEALPix Zarr
in `s3://grid4earth/public/converted/<collection>/` and re-indexes these two categories in the
public STAC index with [stac-scraper](https://github.com/GRID4EARTH/stac-scraper). It is made of
four WorkflowTemplates in `workflows/`:

| WorkflowTemplate | File | Role |
|---|---|---|
| `cdse-orbit-to-eopf-mirror` | `workflows/cdse-orbit-to-eopf-mirror.yaml` | CDSE search of the orbit and selection of the band region (HEALPix parent cells, see below), then one pod per product (at most 4 at a time): copy from the EOPF Sample Service when it has every product of the orbit, else CDSE SAFE download and conversion to EOPF Zarr, into `eopf-mirror/` with the product name as STAC id (complete stores are skipped) |
| `healpix-convert-multistage` | `workflows/healpix-convert-multistage.yaml` | conversion of the orbit strip, clipped to the band region, to one HEALPix Zarr in `converted/` (staging cache, prepare, chunk tasks on 1-CPU pods in balanced lanes, finalize with STAC clean-up) |
| `stac-scraper-update` | `workflows/stac-scraper-update.yaml` | re-indexes only the listed `group/category` paths (the pipeline passes `eopf-mirror/<collection>,converted/<collection>`), leaves the `exclude_stores` out, checks for duplicated item ids, uploads their parquet files then the merged `collections.json` (public-read) |
| `orbit-to-healpix-pipeline` | `workflows/orbit-to-healpix-pipeline.yaml` | checks the arguments and prerequisites, then chains the three templates above with `templateRef` (the STAC step only when `update_stac=true`) |

The pipeline uses three images, each a workflow parameter:

| Parameter | Default | Used for |
|---|---|---|
| `eopf_image` | `s2msi:fefecebf` | SAFE to EOPF Zarr (eopf) |
| `healpix_image` | `g4e-jupyterhub-private:2026-10-01` | CDSE search and band selection (pystac-client, healpix-geo, shapely) |
| `orbit_image` | `g4e-jupyterhub-private:2026-10-10-orbit` | argument check, HEALPix conversion and STAC index (healpix-convert, legacy-converters, stac-scraper) |

### Image build

`orbit_image` is built from `containers/Dockerfile_private_orbit` by the GitHub Actions workflow
**Build orbit image** (`.github/workflows/build-orbit-image.yml`, PR #20) and pushed to
`y74y55mn.gra7.container-registry.ovh.net/healpix-private/g4e-jupyterhub-private:<tag>`, so no
maintainer machine is needed. A `workflow_dispatch` workflow can be started only once it is on the
default branch: after PR #20 is merged, open **Actions → Build orbit image → Run workflow** with
`ref` = `orbit-pipeline` (until this branch is merged, then `main`) and `tag` = `2026-10-10-orbit`
(the repository secrets it needs are listed at the top of the workflow file). The optional inputs
`healpix_convert_sha`, `legacy_converters_sha` and `stac_scraper_sha` override the commits pinned in
the Dockerfile (`HEALPIX_CONVERT_SHA`, ...).

The image must be pushed before the first run. The first step of the pipeline (`check-arguments`)
runs on it: when the image is missing its pod stays in `ImagePullBackOff` and the run fails after
15 minutes (`activeDeadlineSeconds`), before anything is downloaded. Push the image and submit
again. With `orbit_image=<...>:2026-10-01` and `update_stac=false` the pipeline runs without it,
but the conversion then falls back to the converter bundled in the old `legacy_converters` (the
Sentinel-2 `conditions/geometry` and `conditions/meteorology` groups stay empty) and there is no
STAC update. Every pod has a deadline, so a pod that cannot start fails instead of waiting
forever; `stop("<workflow>")` (see [below](#deploying-and-submitting-from-jupyterhub)) stops a
run by hand.

### Region: latitude band snapped to HEALPix parent cells

The orbit is not cut with a bounding box but as a **latitude band** (`lat_range`, `min,max` in
degrees, default `48.0,50.0`, or the preset `france` = `41.3,51.2`) over the **full swath width**,
and the band is snapped to HEALPix **parent cells** of `align_level` (nested, WGS84), so that the
regional products of different orbits line up cell by cell:

1. the orbit is searched in the band widened by two `align_level` cells and its footprints are
   grouped into passes (connected parts, joined when their sensing times are within 5 minutes and
   their longitudes overlap, so that a gap along the track, e.g. a missing row of tiles, does not
   split a pass); when the orbit crosses the band twice (day and night halves, e.g. SLSTR) the
   pass of `reference_item` is kept (manual mode: the pass that intersects `bbox`, which is
   required when there are several passes), and a left-out pass beside the chosen one in
   longitude is reported as a `WARNING`;
2. the footprints of that pass are cut to the band;
3. a cell of `align_level` is kept when it intersects that swath part (`align_rule=intersects`)
   or, with `align_rule=within`, when it also lies entirely inside the band; partial cells along
   the east and west edges of the swath are kept;
4. the union of the kept cells is the region (one polygon; gaps in the swath are filled and their
   cells counted as kept), and the products are those of the pass whose footprint intersects it.

The query passes the region to the conversion as `output_extent` (a GeoJSON polygon drawn through
the cell corners, so that healpix-convert creates the output chunks inside the kept cells) and
`extent_info` (JSON: `lat_range`, `align_level`, `align_rule`, number of cells and actual extent),
stored in the output as the root attribute `orbit_extent`. The query checks the chunks at level 11
(the Sentinel-2 chunk level; coarser chunk levels follow from it) when the kept cells have at most
4^11 children there, and `stage-cache` compares the number of chunks of every group with the
number of cells; both print a `WARNING` when chunks along the cell edges would be added or lost.
This was exact in every case tried with `align_level` 6 or 7 (bands from 49 S to 86 N); with
`align_level` 5 or less, bands near the poles lose some chunks along the cell edges (the warning
says so). The output is clipped
per chunk (level 11 for Sentinel-2, 6 for Sentinel-3, 7 for the Sentinel-3 PSF settings), so
`align_level` must not be finer than the chunk level: `check-arguments` refuses it, and the
default `align_level=7` only fits Sentinel-2 (**use `align_level=6` for Sentinel-3**).

| Parameter | Default | Meaning |
|---|---|---|
| `lat_range` | `48.0,50.0` | latitude band `min,max` or `france` (41.3..51.2); empty or `none` = bounding-box mode (`bbox` is the region, empty = whole orbit, the output is not clipped) |
| `align_level` | `7` | HEALPix level of the parent cells, 0..11 and not above the chunk level of the settings |
| `align_rule` | `intersects` | `intersects`: keep the cells that intersect swath and band; `within`: only the cells entirely inside the band (and intersecting the swath) |
| `bbox` | | in band mode only picks the pass when the orbit crosses the band twice |
| `region_name` | | last part of the output name, see below |

`align_level=7`, `align_rule=intersects` and keeping the partial cells at the swath edges were
confirmed for the first target (decision of 10 October 2026); the default band became 48..50 N
the same day, to halve the conversion. For Sentinel-2C orbit 4025 the default band keeps
**44 level-7 cells** (one polygon, lat 47.49..50.61, lon 0.00..5.89: the cells add ~0.5 degree
north and south, so the 2-degree band covers ~3 degrees, 3-4 rows of MGRS tiles) and **23 of the
25 products** found in the widened band (19.9 GB of SAFE). The former default 45..49 N keeps 77
cells (lat 44.33..49.44, lon -0.77..5.73) and 31 of 44 products (28 tiles: T30TXQ, T30TYQ and
T31TCK have two products at a datatake boundary, both are kept; 25.7 GB); with
`align_rule=within`, 52 cells (lat 45.12..48.66) and 25 products (22.2 GB). The `france` preset keeps 172 cells (lat
40.75..51.77, lon -2.11..6.86) and 71 of 76 products (55.5 GB); the whole orbit has 489 products
(347 GB, lat 3.7..82.8). A pass that crosses the antimeridian is not supported (the query stops
with exit code 2). `legacy-datasets` `safe-to-zarr/download_orbit.py` makes the same selection
with the same defaults (`--lat-range 48.0,50.0 --align-level 7 --align-rule intersects`;
`--lat-range ""` or `none` for bounding-box mode) and the same output name.

**Choosing the orbit by longitude (`lon_range`).** Instead of `reference_item`, give `lon_range`
(`min,max` in degrees) and a search window `datetime` (`start/end` of dates or datetimes, e.g.
`2025-06-08/2025-06-18`; `platform` may limit the satellites). The query then picks **one** orbit
(decision of 10 October 2026; orbits are never combined):

1. the pass whose valid-data footprints cover the largest part of the box `lon_range` x
   `lat_range` (areas in an equal-area projection; coverages within 1 % of the box count as equal);
2. among those, the relative orbit whose swath centre, at the middle latitude of the box, is
   closest to the middle longitude of the box (Sentinel-2A, 2B and 2C share the tracks);
3. among the passes of that track, the one closest to `target_date` (`YYYY-MM-DD` or a datetime;
   empty = middle of the window).

The chosen orbit is then cut as above, over its full swath width, and named as usual (e.g.
`..._R051_O4025_N48-N50`). The query prints the candidates and the coverage of the box, and
`extent_info` gets `orbit_choice` (`lon_range`, window, `target_date`, `coverage`,
`centre_offset_deg` and the first candidates). One orbit may cover the box only in part: a
Sentinel-2 swath is ~290 km wide but slants by ~1.4 degrees of longitude over 45..49 N, so it holds
the whole box there only when the box is about 2.4 degrees wide or less. Over France the tracks
from west to east are R137, R094, R051, R008, R108 (the next track east is 43 relative orbits
lower and passes 3 days earlier). For example, in the default band 48..50 N and the window
2025-06-08..2025-06-18, `lon_range=4.5,5.5` gives Sentinel-2A orbit 52086 (R008, 12 June, 100 %),
and `lon_range=2,5` gives Sentinel-2C orbit 4025 (R051, 98.4 %; the Sentinel-2A and 2B passes of
R051 cover 98.6 %, within 1 %, and 13 June is closest to `target_date=2025-06-13`). In 45..49 N the
box 2..5 is covered ~75 % by both R051 and R008, and R051 wins because its centre is closer:

```python
submit(
    "orbit-to-healpix-pipeline",
    lon_range="2,5",
    datetime="2025-06-08/2025-06-18",
    target_date="2025-06-13",
)
```

**Choosing the orbit for a study region (`study_region`).** Give the id of a region of the
[GRID4EARTH study-regions](https://github.com/GRID4EARTH/study-regions) registry (read from
`https://data.grid4earth.eu/regions.geoparquet`, e.g. `paris`, `mont_blanc`, `rostock1`,
`lake_tuz`) or inline GeoJSON (geometry, Feature or FeatureCollection; its first Feature's
`properties.id` names the output, else give `region_name`). The query then (decision of 10 October
2026):

1. keeps only the passes of the window whose valid data cover the **whole** region (99.9 %); when
   there is none it stops (exit code 2) and prints the best coverage: the region may be wider
   than one swath (~2.4 degrees of longitude for Sentinel-2 at mid-latitudes) or the window too
   short;
2. chooses among them as `lon_range` does: swath centre closest to the middle of the region, then
   the pass closest to `target_date`;
3. sets the latitude band from the region: whole degrees, at least **2 degrees** high (about two
   MGRS tiles of 109.8 km; ~3 degrees and 3-4 tile rows once snapped to the level-7 cells), holding
   the region, its middle closest to the region's (`lat_range` must stay at its default);
4. names the output after the region: its id upper-cased, `_` → `-` (`paris` → `PARIS`,
   `mont_blanc` → `MONT-BLANC`), unless `region_name` is given.

`datetime` defaults to the datetime of the region (a range, or a single date d: d ± 5 days, target
d); `extent_info.orbit_choice` gets `study_region` (`id`, `name`, `bounds`) and the band. In the
window 2025-06-08..2025-06-18: `paris` (48.81..48.90 N) → band 48..50, Sentinel-2C orbit 4025 (R051),
`..._R051_O4025_PARIS` (44 cells, 23 products, as the first target); `mont_blanc` → band 45..47,
Sentinel-2B orbit 43177 (R108, 12 June; only R108 covers it), `..._R108_O43177_MONT-BLANC`;
`rostock1` → band 53..55, Sentinel-2C orbit 4039 (R065), `..._R065_O4039_ROSTOCK1`.

```python
submit(
    "orbit-to-healpix-pipeline",
    study_region="paris",
    datetime="2025-06-08/2025-06-18",
    target_date="2025-06-13",
)
```

**Output name and region tag.** The HEALPix output is
`converted/<collection>/{platform}_{type}_{first sensing start}_R{relative orbit}_O{absolute orbit}_{region}.zarr`.
The region tag is `region_name` when given (letters, digits and `-` only, because the name is
split on `_`; upper-cased, except that a name shaped like a band tag keeps a lower-case `p`:
`n41p3-n51p2` → `N41p3-N51p2`, the tag of that band); otherwise, for a numeric band, the band
itself: hemisphere letter `N` / `S` and absolute degrees per bound, integers without decimals and
`p` for the decimal point of the others (`48.0,50.0` → `N48-N50`, `41.3,51.2` → `N41p3-N51p2`,
`-12.5,-3` → `S12p5-S3`); `FRANCE` for the `france` preset; in bounding-box mode `FRANCE` for the
France bbox, else `REGION`; for a study region its id (see below). Example:
`s3://grid4earth/public/converted/sentinel-2-l2a/S2C_MSIL2A_20250613T104641_R051_O4025_N48-N50.zarr`.

### CDSE credentials

Downloads from CDSE need an account. The workflows read it from the `argo-cdse-credentials`
secret (keys `username` and `password`), mounted read-only at `/etc/cdse`; it is never passed as a
workflow parameter. It is not managed by Tofu (a `tofu apply` would otherwise need the password
on every machine and could overwrite the secret): an admin creates it once with `kubectl`,
prefixing the command with a space or clearing the shell history so that the password is not kept
there:

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
  out (`Retry-After`, in seconds or as an HTTP date; more than 15 minutes stops the product with
  exit code 2, not retried) and the access token is reused for a few minutes. The username and
  password are only posted to an https `*.dataspace.copernicus.eu` token endpoint, and a redirect
  of that request is refused (exit code 2).

### Artifacts

`healpix-convert-multistage` (staging cache and task list) and `stac-scraper-update` (the
`stac-index` output) store their artifacts at an explicit location,
`s3://g4e-desp-argo-artifacts/<template>/<workflow uid>/`, with the `argo-s3-credentials` secret.
They do not rely on the default artifact repository, which the `artifactRepositoryRef` block of
`argo-values.yaml` currently does not define (its key is nested under `data:`, so the
`artifact-repositories` ConfigMap rendered by the chart has no `default-v1-s3` key; check with
`kubectl -n argo get cm artifact-repositories -o yaml`). PR #19 fixes that layout (applied with
`tofu apply -var-file=secrets/terraform.tfvars -target=helm_release.argo_workflows`), which gives
the other workflows a default repository and archived logs; the orbit templates work either way.

### Deploying and submitting from JupyterHub

The Argo server has no public ingress: workflows are submitted from JupyterHub, whose user pods may reach the in-cluster Argo server
service `argo-workflows-server.argo.svc:2746` (network policy `singleuser-to-argo`, plain HTTP).

**Admin, once per change of the templates** (a machine with the cluster kubeconfig, from the
repository root; the pipeline calls the other three with `templateRef`, so they must all be
deployed):

```bash
kubectl apply -n argo \
  -f tf/workflows/cdse-orbit-to-eopf-mirror.yaml \
  -f tf/workflows/healpix-convert-multistage.yaml \
  -f tf/workflows/stac-scraper-update.yaml \
  -f tf/workflows/orbit-to-healpix-pipeline.yaml
```

**User, from a JupyterHub notebook** (or `python` in a JupyterHub terminal), with the Argo
Workflows Python client `argo-workflows`, which the JupyterHub images install
(`containers/Dockerfile_private`; elsewhere `pip install argo-workflows`). The `argo` command-line
tool is not installed in these images and was not tried from a user pod, so the examples in this
README use the Python client. Run this cell once per session:

```python
import json

import argo_workflows
from argo_workflows.api import workflow_service_api
from argo_workflows.model.io_argoproj_workflow_v1alpha1_submit_opts import (
    IoArgoprojWorkflowV1alpha1SubmitOpts,
)
from argo_workflows.model.io_argoproj_workflow_v1alpha1_workflow_retry_request import (
    IoArgoprojWorkflowV1alpha1WorkflowRetryRequest,
)
from argo_workflows.model.io_argoproj_workflow_v1alpha1_workflow_stop_request import (
    IoArgoprojWorkflowV1alpha1WorkflowStopRequest,
)
from argo_workflows.model.io_argoproj_workflow_v1alpha1_workflow_submit_request import (
    IoArgoprojWorkflowV1alpha1WorkflowSubmitRequest,
)

NAMESPACE = "argo"
configuration = argo_workflows.Configuration(host="http://argo-workflows-server.argo.svc:2746")
api = workflow_service_api.WorkflowServiceApi(argo_workflows.ApiClient(configuration))


def call(method, *args, **kwargs):
    """Raw JSON answer: the generated response models are stricter than the server."""
    return json.loads(method(NAMESPACE, *args, _preload_content=False, **kwargs).data or b"{}")


def submit(template, **parameters):
    """Submit a WorkflowTemplate with parameters name=value; returns the workflow name."""
    request = IoArgoprojWorkflowV1alpha1WorkflowSubmitRequest(
        namespace=NAMESPACE,
        resource_kind="WorkflowTemplate",
        resource_name=template,
        submit_options=IoArgoprojWorkflowV1alpha1SubmitOpts(
            parameters=[f"{name}={value}" for name, value in parameters.items()],
        ),
    )
    return call(api.submit_workflow, request)["metadata"]["name"]


def recent(count=5):
    """(name, phase) of the latest workflows."""
    fields = "items.metadata.name,items.metadata.creationTimestamp,items.status.phase"
    items = call(api.list_workflows, fields=fields).get("items") or []
    items.sort(key=lambda w: w["metadata"]["creationTimestamp"], reverse=True)
    return [(w["metadata"]["name"], w.get("status", {}).get("phase")) for w in items[:count]]


def status(name):
    """(phase, progress, message, failed steps) of a workflow."""
    st = call(api.get_workflow, name).get("status", {})
    failed = sorted(node.get("displayName", "") for node in (st.get("nodes") or {}).values()
                    if node.get("type") == "Pod" and node.get("phase") in ("Failed", "Error"))
    return st.get("phase"), st.get("progress"), st.get("message"), failed


def logs(name, pod_name=None, grep=None):
    """Print the logs of a workflow, or of one of its pods, optionally only matching lines."""
    options = {key: value for key, value in (("pod_name", pod_name), ("grep", grep)) if value}
    response = api.workflow_logs(NAMESPACE, name, log_options_container="main",
                                 _preload_content=False, **options)
    for line in response.data.decode().splitlines():
        entry = json.loads(line).get("result", {})
        print(entry.get("podName", ""), entry.get("content", ""))


def retry(name):
    """Run the failed steps of a finished, failed workflow again (the others are kept)."""
    request = IoArgoprojWorkflowV1alpha1WorkflowRetryRequest(name=name, namespace=NAMESPACE)
    return call(api.retry_workflow, name, request)["metadata"]["name"]


def stop(name):
    """Stop a running workflow (its running steps are stopped, exit handlers still run)."""
    request = IoArgoprojWorkflowV1alpha1WorkflowStopRequest(name=name, namespace=NAMESPACE)
    return call(api.stop_workflow, name, request)["metadata"]["name"]
```

Then, for example:

```python
name = submit(
    "orbit-to-healpix-pipeline",
    reference_item="S2C_MSIL2A_20250613T104641_N0511_R051_T31UDQ_20250613T134507",
)
status(name)               # phase, progress, message, failed steps
logs(name, grep="ERROR")   # or logs(name, pod_name=...) for one step
recent()                   # the latest workflows, if the name was lost
```

Other parameters of a template are further keyword arguments of `submit` (`name=value` in this
README).

### First target: Sentinel-2C orbit 4025, 48.0..50.0 N, every group

The first production run (decision of 10 October 2026) is relative orbit 51, absolute orbit 4025
of Sentinel-2C (13 June 2025), identified by one of its products, with the defaults: latitude band
48.0..50.0 N (45.0..49.0 N at first, narrowed the same day to halve the conversion), full swath width, level-7 parent cells, `align_rule=intersects`, partial east / west
edge cells kept, **all conversion groups** (`groups` empty):

```python
submit(
    "orbit-to-healpix-pipeline",
    reference_item="S2C_MSIL2A_20250613T104641_N0511_R051_T31UDQ_20250613T134507",
)
```

The 23 products of the band region are mirrored in `s3://grid4earth/public/eopf-mirror/sentinel-2-l2a/`
and the HEALPix result is written to
`s3://grid4earth/public/converted/sentinel-2-l2a/S2C_MSIL2A_20250613T104641_R051_O4025_N48-N50.zarr`.
`study_region="paris"` (below) picks the same orbit and band for the 13 June window and names the
output `..._R051_O4025_PARIS.zarr`.
Before it, the bounding-box test below checks the image, the secrets and the cluster on 2
products, and `stac-scraper-update` with `dry_run=true` on the two categories finds index
problems that would otherwise only stop the run after the conversion (see "STAC index").

Main parameters of `orbit-to-healpix-pipeline` besides the region ones above (the others are
described in the template):

| Parameter | Default | Meaning |
|---|---|---|
| `collection` | `sentinel-2-l2a` | CDSE collection (`sentinel-2-l1c`, `sentinel-2-l2a`, `sentinel-3-olci-1-efr-ntc`, `sentinel-3-sl-1-rbt-ntc`, ...) |
| `reference_item` | | one CDSE product of the orbit; otherwise give `datetime` (and `platform`, `absolute_orbit`, `relative_orbit`), not both |
| `lon_range` | | `min,max`: choose one orbit in the `datetime` window by its coverage of `lon_range` x `lat_range` (see above); leave `reference_item`, `relative_orbit` and `absolute_orbit` empty |
| `study_region` | | id of a study region (e.g. `paris`) or inline GeoJSON: choose one orbit that covers the whole region; sets the band (>= 2 degrees) and the output name (`PARIS`), see above |
| `target_date` | | with `lon_range` / `study_region`: among passes of equal coverage on the same track, the one closest to this date (empty = middle of `datetime`, or the date of the study region) |
| `check_sample_service` | `true` | copy the products from the EOPF Sample Service when it has every product of the orbit (stores from both sources cannot be merged into one HEALPix dataset) |
| `settings_name` | `sentinel-2-l2a` | `legacy_converters` settings, named after the G4E collection: it **must match** `collection` (empty = derived from it); the run stops at the argument check otherwise |
| `groups` | all | JSON list of groups to convert, e.g. `'["measurements/reflectance/r60m"]'` |
| `convert_parallelism` | `48` | conversion pods (1 CPU; 4 Gi requested / 8 Gi limit, 7 Gi / 10 Gi for the 10 m nearest groups) running at the same time |
| `max_tasks` | `500` | upper bound on the number of conversion pods of the run |
| `chunks_per_task` | `16` | minimum number of output chunks per conversion pod (raised per group, see below) |
| `nworkers` | `1` | dask workers per conversion pod; the pods have 1 CPU, keep 1 |
| `overwrite` | `false` | replace an existing HEALPix output |
| `update_stac` | `true` | re-index `eopf-mirror/<collection>` and `converted/<collection>` after the conversion (needs the default `mirror_prefix` / `converted_prefix` and a stac-scraper collection template, see below) |
| `stac_dry_run` | `false` | build the STAC index and print the uploads without publishing |

For Sentinel-3, set `settings_name` to the G4E collection (or its `-psf` variant) and
`align_level` to 6 (7 for the `-psf` settings), e.g. for OLCI EFR:

```python
submit(
    "orbit-to-healpix-pipeline",
    collection="sentinel-3-olci-1-efr-ntc",
    settings_name="sentinel-3-olci-l1-efr",
    align_level=6,
    reference_item="S3B_OL_1_EFR____20250613T111915_20250613T112215_20250614T123815_0179_107_308_2160_ESA_O_NT_004",
)
```

which writes
`s3://grid4earth/public/converted/sentinel-3-olci-l1-efr/S3B_OL_1_EFR_20250613T111615_R308_O37155_N48-N50.zarr`
(2 products, 61 level-6 cells, lon -24.5..-3.3: this pass lies west of France; in 45..49 N it
had 1 product and 86 cells; with
`lat_range=france` it has 3 products and 173 cells and is named `..._FRANCE.zarr`). The STAC
update of `eopf-mirror/sentinel-3-olci-l1-efr` works because the template leaves the second copy
of an older OLCI product out of the index (see `exclude_stores` below).

For a first test, keep both the mirror and the HEALPix output in `tmp/` (never indexed), use
bounding-box mode on a small area (`lat_range=none`, or empty; here 2 products around Paris) and
one group, without touching the STAC index:

```python
submit(
    "orbit-to-healpix-pipeline",
    reference_item="S2C_MSIL2A_20250613T104641_N0511_R051_T31UDQ_20250613T134507",
    lat_range="none",
    bbox="2.2,48.7,2.5,48.9",
    region_name="PARIS",
    mirror_prefix="s3://grid4earth/public/tmp/eopf-mirror",
    converted_prefix="s3://grid4earth/public/tmp/converted",
    groups='["measurements/reflectance/r60m"]',
    update_stac="false",
)
```

### Scale, wall time and bottlenecks

Measured for the first target in its first band (orbit 4025, 45.0..49.0 N, level 7; the default
band 48..50 N has 44 cells, 23 products and 11 264 chunks per group, 57 % of it): 77 cells, 31 products (28
tiles), 25.7 GB of SAFE; 19 712 level-11 output chunks per chunked group in the extent, 15 609 of
which hold data (by the footprints of the 31 products; 503 of them cross the edge of the data).
Converting one chunk of all 16 chunked Sentinel-2 L2A groups takes 13.3 s locally with 4 threads
and ~22 s with 1 (10 m reflectances: 4.85 s and 6.0 s), so four 1-CPU pods convert more than
twice as much as one 4-CPU pod: the conversion runs on **many 1-CPU pods** (one thread each:
`OMP_NUM_THREADS`, `MKL_NUM_THREADS`, `OPENBLAS_NUM_THREADS`, `NUMEXPR_NUM_THREADS`,
`RAYON_NUM_THREADS` = 1 and `torch.set_num_threads(1)`). Their memory depends on the group (pod
classes, see below): "standard" pods request 4 Gi with an 8 Gi limit (about 14 per b3-64 node of
16 vCPU / 64 GB), "large" pods 7 Gi with a 10 Gi limit (about 8 per node).

Memory: converting a chunk allocates and frees up to ~1.7 GiB (10 m `nearest` groups; 1.1 GiB for
the 10 m reflectances, less than 0.9 GiB for the other groups) and nothing accumulates from one
chunk to the next (Python objects, tracemalloc and the bytes in use of malloc stay flat), but the C
allocator keeps what each chunk frees: converted in a single process, a task of 472 chunks of a
10 m mask reached 8.0 GiB resident locally (macOS). Three measures bound it:

- after a chunk (at most every 2 s), `convert-chunk` hands the freed memory back to the system
  (glibc `malloc_trim(0)`, macOS `malloc_zone_pressure_relief`, ~0.05 s each);
- the tasks of the 10 m and 20 m groups are converted in sub-batches of 2.5-5 minutes of work
  (`CHUNKS_PER_PROCESS`: 50 chunks of a 10 m group, 80 of a 20 m reflectance, 400 of a 20 m
  mask), each in a new process that gives all its memory back when it ends; the start-up of a
  process (~5-7 s locally) costs 2-4 % of its work;
- two pod classes, both 1 CPU: the three 10 m `nearest` groups (`quality/mask/r10m`,
  `quality/atmosphere/r10m`, `conditions/mask/detector_footprint/r10m`) cannot stay under 4 GiB
  (a fresh process reaches ~3 GiB in its first chunk: the resampler's per-chunk transient, see
  healpix-resample#69), so they run in **large** pods (7 Gi requested, 10 Gi limit); every other
  group runs in **standard** pods (4 Gi requested, 8 Gi limit). `stage-cache` writes the class
  and its memory into each task (`convert-chunk` sets it with `podSpecPatch`) and gives each class
  lanes of its own, so the memory requested by the running pods stays fixed.

Peak resident memory per process, measured locally (macOS, one thread, with the release above;
the parent process of the sub-batches stays at ~15 MB):

| Group(s) | Pod class | Chunks per process | Peak RSS of a process |
|---|---|---|---|
| 10 m `nearest` (masks, atmosphere, detector footprint) | large (7 / 10 Gi) | 50 | 3.9-4.2 GiB (5.0 GiB for 100 chunks; 6.3 GiB for 50 without the release) |
| 10 m reflectances (`psf`) | standard (4 / 8 Gi) | 50 | 2.1-2.3 GiB, also at the edge of the data (3.3 GiB without the release) |
| 20 m `nearest` (masks, probability, classification) | standard | 400 | 2.8 GiB |
| 20 m reflectances (`psf`) | standard | 80 | ~1.2 GiB (40 chunks, without the release) |
| 60 m and `no_chunk` groups | standard | whole task | < 1 GiB |

How much glibc keeps on the cluster nodes, and how well `malloc_trim` hands it back, has not been
measured: each `convert-chunk` log ends with the peak RSS of its processes. If one comes close to
its limit, lower `CHUNKS_PER_PROCESS` for that group in the template, or move its key to
`LARGE_POD_KEYS`.

Estimate: **~160-165 CPU-hours** on the cluster (cores assumed 1.5x slower than the benchmark
machine; ~8 of them for the start-up of the ~2 000 sub-batch processes), ~150 GB of output in a
few 100 000 to ~1 million objects. Gridlook cannot display such a strip at full resolution: the
output holds only the data levels (20 / 19 / 17 for 10 / 20 / 60 m), so a coarse level has to be
provided for visualisation (for example a separate coarsening step).

The large pods carry ~39 % of the work, so their lanes are about 40 % of `convert_parallelism`:

| Conversion pods (large + standard lanes) | Memory requested | b3-64 nodes | Mean lane | Conversion wall time (longest lane) |
|---|---|---|---|---|
| 32 (13 + 19) | ~167 Gi | 3 | ~5.1 h | ~5.8 h |
| **48 (19 + 29, default)** | ~249 Gi | 5 | ~3.4 h | ~3.9 h |
| 64 (24 + 40) | ~328 Gi | 6 | ~2.5 h | ~3.3 h |
| 8 x 4 CPU / 16 Gi (former default) | 128 Gi | 2-3 | | ~11 h |

The longest lane is ~1.15x the mean with 32 or 48 pods and ~1.3x with 64 (simulated with the
footprints of the 31 products): the tasks are sized and balanced with every chunk counted as full,
but the ~21 % of chunks without data (east and west swath edges) cost almost nothing and gather in
some tasks, since a task is a range of consecutive chunks. Add the mirror stage (~40-60 min) and a few minutes each for the staging
cache, `prepare`, `finalize` and the STAC update. The `cpu-workers` and
`dask-workers` node pools (b3-64, autoscaling 1..5 each, both untainted, so Argo pods land on both:
at most 10 nodes) are shared with JupyterHub and Dask Gateway; `convert_parallelism=48` needs about
5 of them, `convert_parallelism=64` about 6.

How the conversion is split: `stage-cache` makes at most `max_tasks` tasks. The groups cost very
different times per chunk (one thread: 6 s for the 10 m reflectances, 3 s for a 10 m mask, 0.06 s
for a 60 m mask), so the chunks per task are chosen per group from a benchmark table
(`CHUNK_COST_S` in the template) for tasks of about the same time (~16 min at benchmark speed for
the first target: 156 chunks of a 10 m reflectance group, 15 581 of a 60 m mask), at least
`chunks_per_task`. A task of a 10 m or 20 m group is converted in sub-batches of 2.5-5 minutes of
work (`CHUNKS_PER_PROCESS`: 50 chunks of a 10 m group, 80 of a 20 m reflectance, 400 of a 20 m
mask), each in a new process (see the memory paragraph above). The tasks are then dealt into
`convert_parallelism` lanes (Argo cannot template `parallelism`): each pod class gets lanes of its
own (the split with the shortest longest lane: 19 large + 29 standard for 48 lanes), and within a
class the tasks go longest first to the least loaded lane; each lane runs its tasks one after
another. For the first target with 48 lanes the longest lane is 1.06x the mean in estimated times
(1.49x for the former round-robin of equal batches); with the real work (chunks without data
almost free) ~1.15x, as in the table above. The `stage-cache` log prints the plan, the lanes and
memory per pod class, and an upper estimate (chunks without data counted as full: 200 CPU-hours,
longest lane 4.4 h, longest task 0.4 h for the first target with 48 lanes).

Bottlenecks:

- **CDSE downloads**: 4 concurrent connections per account, so the mirror stage runs at most 4
  pods (~40-60 min for the 31 products); more pods would only get HTTP 429 answers.
- **Staging cache**: `stage-cache` (and `prepare`) are single pods that open every input store
  (cache.json ~50 KB per input, ~1.6 MB for 31 inputs; ~0.5 GiB expected for 31 inputs, 2 CPU /
  8 Gi given), and every conversion process (one per sub-batch) re-opens the 31 inputs from the
  cache, part of its start-up (~5-7 s measured with one local input, of 2.5-5 minutes of work per
  process; not measured with 31 inputs on S3, where the `set-up` time in the log shows it).
- **S3 object count**: one object per chunk and array gives a few 100 000 to ~1 million objects
  per orbit, which slows uploads, the listing in `finalize` and any later copy or deletion. A
  future option is Zarr sharding with one shard per level-7 parent cell (256 level-11 chunks),
  which lines up with the region cells; it needs every shard to be written by a single task.
- **Cluster size**: 48 pods (19 large, 29 standard) request ~249 Gi, about 5 of the 10 shared nodes.

### Timeouts and retries

The pipeline is meant to survive slow links (a few MB/s) and proxy errors (502, timeouts):

| Step | Retries | Deadline per attempt |
|---|---|---|
| `query` | 2, backoff 30 s x2 | 30 min |
| `mirror-item` | 4, backoff 2 min x2, no retry started after 2 h | 2 h (sized for 3 full downloads of a 1.3 GB SAFE at 2 MiB/s, conversion and upload) |
| `stage-cache`, `prepare` | 2, backoff 30 s x2 (`stage-cache`: also exit code 3 = output exists) | 2 h |
| `convert-chunk` | 4, backoff 1 min x2 (not for the resampler error "No HEALPix cell passed the threshold", which would repeat) | 4 h (a task of the first target: ~0.4 h, at most ~0.5 h) |
| `finalize` | 2, backoff 30 s x2 | 1 h |
| `stac-scraper-update` | 3, backoff 1 min x2 | 2 h |

Exit code 2 (invalid arguments or data, duplicated ids, missing credentials) is never retried.

- HTTP: STAC searches use a (60 s connect, 600 s read) timeout and retry 429 / 5xx; CDSE
  downloads use the same timeouts, wait out 429 / 503 (`Retry-After`) and restart from zero
  (CDSE answers 416 to range requests), with a checksum check.
- S3 (s3fs / botocore, also in the dask workers through `FSSPEC_S3`): connect timeout 120 s, read
  timeout 600 s, up to 10 adaptive retries; stac-scraper (obstore) gets the same timeouts.
- Image build: `pip install --timeout 120 --retries 10` in `containers/Dockerfile_private_orbit`.

### STAC index

- The HEALPix dataset gets its root `stac_discovery`, which makes stac-scraper index it, only in
  the last conversion step (`finalize`), once every chunk is written; until then the item is kept
  as `stac_discovery_pending`. With `overwrite=true` the existing dataset is deleted at the start
  of the conversion and is not indexed again before `finalize`.
- healpix-convert copies the properties of the *first* input product (one tile) into the item.
  `prepare` therefore sets the time range of all inputs and `created` (processing time), and
  drops the properties whose value differs between the inputs (tile-specific ones such as
  `eopf:datastrip_id`), except `eo:cloud_cover` and `eo:snow_cover`, which become the mean over
  the input products (not weighted by area, and over whole tiles, not only the band).
- `finalize` cleans the item before publishing it: no `proj:*` properties (the output is not in
  UTM), no assets (stac-scraper adds its own), no self / root / parent / collection links,
  `derived_from` links and `processing:lineage` keep public URLs only
  (`s3://grid4earth/public/<key>` becomes `https://data.grid4earth.eu/<key>`, other paths are
  dropped), and `geometry` / `bbox` are the union of the chunk cells that hold data.
  `processing:expression` (the conversion settings, an object) is moved to the root attribute
  `healpix_convert_settings` of the Zarr: the catalogue's other items have the string
  `systematic`, the geoparquet writer fails on mixed types, and the processing extension schema
  requires an object, so the item leaves the field out.
- `update-stac` re-indexes only `eopf-mirror/<collection>` and `converted/<collection>` (passed
  as `include_categories`, from the collection of the query's `orbit` output); every other
  category is excluded, so a broken store elsewhere in the bucket cannot block the update. Only
  these two parquet files are uploaded, then `collections.json`, which is **merged**: the
  published file is read, the two collections (e.g. `mirror-sentinel-2-l2a` and
  `converted-sentinel-2-l2a`) are replaced in place or appended, every other entry is kept as it
  is, and nothing is ever removed. A diff (added / replaced / kept) is printed.
- **Item ids.** The ids that eopf (3.0 / rc4) gives the tiles of one Sentinel-2 datatake differ
  only by a 3-hex-digit CRC (4096 values): the chance of a duplicate is about 6 % among the 23
  products of the first target (48.0..50.0 N; 11 % among the 31 of 45.0..49.0 N) and 46 % among the 71 of the `france` preset, and
  the STAC API answers 404 for every item of a duplicated id. `mirror-item` therefore sets the `stac_discovery` id of each new store to the
  product name (the store name without `.zarr` / `.SEN3` / `.SAFE`, e.g.
  `S2C_MSIL2A_20250613T104641_N0511_R051_T31UDQ_20250613T134507`, the id stac-scraper's
  `--id-from-store-name` of stac-scraper PR #24 gives) and keeps the eopf id in the root
  attribute `g4e_mirror.eopf_stac_id`. Stores mirrored before keep their eopf id.
- `exclude_stores` (store paths relative to `s3://grid4earth/public`, or URLs) leaves stores out
  of the index while they stay in the bucket. The template always adds the second copy of the
  OLCI product of 15 January 2026 (`eopf-mirror/sentinel-3-olci-l1-efr/..._004.zarr`; the
  `..._004.SEN3.zarr` copy is the one catalogued, decision of 9 October 2026). With stac-scraper
  62b5c01, which has no `--exclude-stores`, their items are removed from the scraped parquet file;
  a stac-scraper with the option (PR #24) is given it.
- The update stops before uploading anything (exit code 2, not retried) when a target parquet
  file still has **duplicated item ids** (e.g. two stores mirrored before the id change): the ids
  and their stores are printed; add one store of each pair to `exclude_stores` (or give it a
  distinct id) and run `stac-scraper-update` again (below). It also stops on a dataset directory
  without a root `zarr.json` (an interrupted upload: delete it), on a store stac-scraper cannot
  index (e.g. a `processing:expression` object written before this clean-up: the store is
  named), on a target collection without a stac-scraper collection template, and on a
  `dry_run` other than `true` / `false`. These checks run only after the conversion: to find
  such problems before a long run, run `stac-scraper-update` with `dry_run=true` on the two
  categories first.
- stac-scraper 62b5c01 has no collection template for `sentinel-3-slstr-l2-frp` and
  `sentinel-3-slstr-l2-lst`: `check-arguments` refuses `update_stac=true` for them, run these
  collections with `update_stac=false`. `update_stac=true` also requires the default
  `mirror_prefix` / `converted_prefix` (the indexed categories under `s3://grid4earth/public`).

The STAC index can also be refreshed on its own. `include_categories` takes exact
`group/category` paths (a bare group such as `converted` is refused); `exclude_stores` takes
store paths; with `dry_run=true` nothing is uploaded and the generated index is kept as the
`stac-index` output artifact:

```python
submit(
    "stac-scraper-update",
    include_categories="eopf-mirror/sentinel-2-l2a,converted/sentinel-2-l2a",
    exclude_stores="eopf-mirror/sentinel-2-l2a/<second copy>.zarr",
    dry_run="true",
)
```

### Behaviour worth knowing

- Re-running on the same orbit skips the products already mirrored (a store is complete when its
  root `zarr.json` has `stac_discovery`; an incomplete one left by a failed attempt is deleted and
  written again), but the conversion refuses an existing output unless `overwrite=true`. To
  resume a run whose conversion tasks failed, use `retry("<workflow>")` (an Argo retry, see
  [above](#deploying-and-submitting-from-jupyterhub); only the failed tasks run again; a failed
  task does not stop the other tasks of its lane).
- Groups whose inputs cannot be concatenated are left out with a warning in the `stage-cache` log:
  for a Sentinel-2 orbit, `conditions/geometry`, whose `detector` dimension differs between full
  and partial tiles.
- `healpix-convert-multistage` works around issues of healpix-convert 3aa72e0 until they are
  fixed upstream (see the comment at the top of the template): input STAC items with different
  `stac_extensions` are merged as a union; a chunk without any input point, and for the
  `nearest` resampler (Sentinel-2 quality masks, detector footprints, classification) a chunk
  whose input points all lie in its buffer zone outside the chunk cell (along the outer edge of
  the tiles), is left empty instead of failing in the resampler (upstream fix:
  healpix-convert#25, which closes #23); and the resamplers get float64 coordinates (the float32
  ones of the `no_chunk` groups make them fail).
- **Half-pixel shift (temporary workaround).** For EOPF variables without `proj:*` attributes
  (the eopf 3.0 SAFE to Zarr output), healpix-convert 3aa72e0 takes the pixel-*center* transform
  of the x/y coordinates as the pixel-*corner* transform, which shifts the output half a source
  pixel east and south (5 / 10 / 30 m at 10 / 20 / 60 m). Every stage of
  `healpix-convert-multistage` wraps `extract_spatial_info_stac` (`FIX_HALF_PIXEL`) to return the
  corner transform; the wrapper does nothing when the library already returns it. The upstream
  fix is healpix-convert#24; the workaround stays in the template until it is released, then bump
  `HEALPIX_CONVERT_SHA` in `containers/Dockerfile_private_orbit` (or pass
  `healpix_convert_sha` to the image build), rebuild `orbit_image`, and the workaround becomes a
  no-op (it can then be removed).
  HEALPix outputs made from such inputs with an unpatched healpix-convert are shifted and should
  be converted again.
- Objects written under `s3://grid4earth/public/` are public-read, like the rest of that area.

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
