# Google Cloud Console + GitHub UI + Argo CD UI (Sinhala Guide)

මෙම guide එක CLI commands අඩු කරලා, හැකි තරම් Web dashboards හරහා setup කරන පියවර-පියවර ක්‍රමයයි.

ඉලක්කය:

- ඔබ main branch එකට push කරන හැම වෙලාවෙම Docker images auto rebuild වෙන්න
- image tags auto update වෙන්න
- Argo CD මගින් GKE cluster එක auto sync වෙලා app එක update වෙන්න

---

## 1) Overall flow එක (සරල map එක)

1. ඔබ GitHub වෙත code push කරනවා
2. GitHub Actions workflow run වෙලා image එක Google Artifact Registry වෙත push කරනවා
3. workflow එක repo එකේ `newfolder/k8s/overlays/prod/kustomization.yaml` update කර commit කරනවා
4. Argo CD එම change එක detect කර GKE cluster එකට auto sync කරනවා
5. Kubernetes rolling update එකෙන් new pods run වෙනවා

---

## 2) Google Cloud Project + Billing prepare කරන්න (Dashboard only)

1. Google Cloud Console login කරන්න
2. top project selector එකෙන් New Project click කරන්න
3. Project name ලෙස `predictive-maintenance-prod` වගේ නමක් දාන්න
4. Billing account එක project එකට link කරන්න

Note:

- Billing enable නොවුණොත් GKE/Artifact Registry resources create වෙන්නේ නැහැ

---

## 3) Required APIs enable කරන්න (Console)

Google Cloud Console -> APIs & Services -> Enable APIs and Services:

1. Kubernetes Engine API
2. Artifact Registry API
3. IAM Service Account Credentials API
4. Compute Engine API

Tip:

- API enable වෙලා තත්පර/මිනිත්තු කිහිපයක් wait කරලා පසුව resource creation කරන්න

---

## 4) Artifact Registry create කරන්න (Docker images සඳහා)

1. Console -> Artifact Registry -> Repositories
2. Create Repository click කරන්න
3. Format: Docker
4. Name: `predictive-maintenance`
5. Region: ඔබගේ GKE region එකට same region (උදා: `us-central1`)
6. Create click කරන්න

Image path format:

- `<REGION>-docker.pkg.dev/<PROJECT_ID>/<REPOSITORY>/<IMAGE_NAME>:<TAG>`
- Example: `us-central1-docker.pkg.dev/my-project/predictive-maintenance/predictive-maintenance-frontend:latest`

---

## 5) GKE Cluster + Nodes create කරන්න (Dashboard)

1. Console -> Kubernetes Engine -> Clusters
2. Create click කරන්න
3. Standard (හෝ Autopilot) cluster type තෝරන්න
4. Cluster name: `predictive-maintenance-gke`
5. Region/Zone ඔබගේ latency budget එකට ගැළපෙන එක තෝරන්න
6. Node pool:
   - Machine family: workload එකට ගැළපෙන එක
   - Node count: අවම 2 (HA learning path සඳහා)
7. Create click කරන්න

Nodes ready check:

1. cluster details page -> Nodes tab
2. nodes status Ready/Running ද කියලා verify කරන්න

---

## 6) Artifact Registry image pull permission align කරන්න (UI)

GKE nodes image pull කරන්න Artifact Registry read permission තියෙන්න ඕන.

Dashboard path:

1. Console -> Kubernetes Engine -> Clusters -> ඔබගේ cluster -> Details
2. Node service account (Compute Engine default SA හෝ custom SA) note කරගන්න
3. Console -> IAM & Admin -> IAM
4. එම service account එකට role add කරන්න:
   - `Artifact Registry Reader`

වැදගත්:

- GKE cluster සහ Artifact Registry same project එකේ නම් hardcoded `imagePullSecrets` බොහෝ අවස්ථාවල අවශ්‍ය නැහැ
- old registry secret names (උදා: DO specific values) manifests වල තිබ්බොත් remove කරන්න

---

## 7) Argo CD install කරන්න (Web dashboard path)

Option A (recommended): Google Cloud Marketplace හරහා deploy

1. Console -> Marketplace
2. `Argo CD` search කරන්න
3. GKE deployment option select කරන්න
4. Cluster: `predictive-maintenance-gke`
5. Namespace: `argocd`
6. Service exposure (LoadBalancer/Ingress) enable කර UI access allow කරන්න
7. Deploy click කරන්න

Install පසු:

1. Kubernetes Engine -> Workloads/Services වලින් Argo CD server external endpoint ගන්න
2. browser එකෙන් Argo CD UI open කරන්න

Tip:

- Marketplace form එකේ admin password set option තියෙනවා නම් setup වෙලාවෙම set කරන්න

---

## 8) Argo CD UI එකෙන් Application create කරන්න

Argo CD UI -> New App:

1. Application Name: `predictive-maintenance`
2. Project: `default`
3. Sync Policy: Automatic
4. Prune Resources: Enable
5. Self Heal: Enable
6. Repository URL: ඔබගේ GitHub repository URL
7. Revision: `main`
8. Path: `newfolder/k8s`
9. Destination Cluster: `https://kubernetes.default.svc`
10. Destination Namespace: `predictive-maintenance`
11. Create App click කරන්න

Result:

- app එක Synced + Healthy status එකට යා යුතුයි

---

## 9) GitHub Web UI එකෙන් Actions Secrets set කරන්න

GitHub repo -> Settings -> Secrets and variables -> Actions -> New repository secret

### A) Service Account Key (simple path)

Required secrets:

1. Name: `GCP_PROJECT_ID`
2. Value: ඔබගේ GCP project id

1. Name: `GCP_REGION`
2. Value: උදා: `us-central1`

1. Name: `GAR_REPOSITORY`
2. Value: `predictive-maintenance`

1. Name: `GCP_SA_KEY`
2. Value: service account JSON key content (single secret value එකක් ලෙස)

Service account role suggestions (minimum):

- Artifact Registry Writer
- (optional) Viewer

### B) Workload Identity Federation (more secure path)

Required secrets:

1. `GCP_PROJECT_ID`
2. `GCP_REGION`
3. `GAR_REPOSITORY`
4. `GCP_WORKLOAD_IDENTITY_PROVIDER`
5. `GCP_SERVICE_ACCOUNT`

Note:

- Production සඳහා JSON key වලට වඩා WIF path එක recommended

---

## 10) Kustomize image paths GAR format එකට set කරන්න

`newfolder/k8s/overlays/prod/kustomization.yaml` file එකේ `images.newName` values GAR path වලට set කරන්න.

Example format:

```yaml
images:
  - name: predictive-maintenance-frontend
    newName: us-central1-docker.pkg.dev/my-project/predictive-maintenance/predictive-maintenance-frontend
    newTag: latest
```

ඔබගේ services සියල්ලටම same pattern භාවිතා කරන්න:

- frontend
- api-gateway
- ingestion
- event-processing
- ml
- notification

---

## 11) GitHub Actions workflow verify කරන්න (Web UI)

check කරන්න:

- `.github/workflows/build-and-deploy.yml`

workflow එක GCP login + Artifact Registry push support කරනවාද කියලා verify කරන්න.

Expected behavior:

1. Docker images build/push to Artifact Registry
2. `newfolder/k8s/overlays/prod/kustomization.yaml` tag update
3. updated manifest commit/push

---

## 12) First test (CLI නැතුව end-to-end)

1. GitHub web editor එකෙන් small commit එකක් main branch ට දාන්න
2. Actions tab එකේ workflow run success ද බලන්න
3. bot commit (`ci: set images ...`) history එකට ඇවිත් තියෙනවාද බලන්න
4. Argo CD UI එකේ OutOfSync -> Synced/Healthy transition verify කරන්න
5. GKE Workloads view එකේ new rollout verify කරන්න

---

## 13) Nodes scale/registration (Web UI)

1. Console -> Kubernetes Engine -> Clusters -> Node pools
2. node pool edit කර node count වැඩි/අඩු කරන්න
3. Save click කරන්න
4. Nodes tab එකේ new nodes Ready status verify කරන්න

---

## 14) UI-based Troubleshooting (GCP specific)

### A) Actions fail with authentication error (GAR push)

Issue:

- Logs වල `denied: Permission "artifactregistry.repositories.uploadArtifacts" denied` වගේ error.

Fix:

1. GitHub භාවිතා කරන service account එකට `Artifact Registry Writer` role add කරන්න
2. secret values typo නැද්ද verify කරන්න (`GCP_PROJECT_ID`, `GCP_REGION`, `GAR_REPOSITORY`)

### B) Pod `ImagePullBackOff` from Artifact Registry

Issue:

- GKE pod image pull කරන්නේ නැහැ.

Fix:

1. Node service account එකට `Artifact Registry Reader` role තියෙනවාද verify කරන්න
2. image path region/repository/project id mismatch නැද්ද බලන්න
3. old `imagePullSecrets` values තිබ්බොත් remove කර GKE default auth flow use කරන්න

### C) Argo CD app create කරන වෙලාවේ `kustomization.yaml is empty`

Issue:

- `newfolder/k8s/overlays/prod/kustomization.yaml` empty/invalid.

Fix:

1. file content valid Kustomize syntax ද කියලා check කරන්න
2. Argo App path `newfolder/k8s` ලෙස set කරන්න
3. `newfolder/k8s/kustomization.yaml` -> `overlays/prod` reference confirm කරන්න

### D) `namespace not found` sync failure

Fix:

1. `newfolder/k8s/base/namespace.yaml` file resource list එකට include වී ඇතිද බලන්න
2. Argo app destination namespace `predictive-maintenance` සමඟ match වෙනවාද verify කරන්න

### E) Ingress external IP pending

Issue:

- Ingress/LoadBalancer external IP assign වෙන්න delay.

Fix:

1. cluster region quota and external IP quota check කරන්න
2. ingress class/controller running ද verify කරන්න
3. firewall/network policy restrictions බලන්න

### F) `FailedAttachVolume` / Multi-Attach (RWO PVC)

Fix:

- single replica + RWO PVC workloads සඳහා deployment strategy `Recreate` consider කරන්න

---

## 15) Learning checklist

- [ ] GCP project + billing ready
- [ ] Artifact Registry create කරලා තියෙනවා
- [ ] GKE cluster + nodes create කරලා තියෙනවා
- [ ] Node SA -> Artifact Registry Reader role assign කරලා තියෙනවා
- [ ] Argo CD install + App create කරලා තියෙනවා
- [ ] GitHub Actions GCP secrets set කරලා තියෙනවා
- [ ] main push එකකින් auto build + auto deploy වැඩ කරනවා

---

## 16) GCP VPC තුළ run වෙන app access කරන ක්‍රම

වැදගත්:

- VPC එක network boundary එකක්. direct "VPC console" එකක් නැහැ.
- access method එක workload type එක මත වෙනස් වේ.

### A) GKE workload logs (UI)

1. Console -> Kubernetes Engine -> Workloads
2. relevant deployment/pod click කරන්න
3. Logs tab එකෙන් runtime logs බලන්න

### B) Pod shell access (UI-assisted)

1. Console -> Kubernetes Engine -> Workloads -> Pod details
2. Exec/Terminal option available නම් shell open කරන්න

### C) VM based component access

1. Console -> Compute Engine -> VM instances
2. target VM -> SSH button (browser terminal)

Note:

- managed Kubernetes nodes වල manual node-level changes persistent නැති විය හැක
- troubleshooting සඳහා pod/deployment layer access එක preferred

---

මෙය dashboard-first GitOps learning path එකකි. advanced debugging සඳහා පමණක් CLI path එක භාවිතා කරන්න.
