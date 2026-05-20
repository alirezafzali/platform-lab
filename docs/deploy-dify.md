# Deploy Dify on staging (VKE + Argo CD)

Dify is deployed via the [dify-helm](https://github.com/BorisPolonsky/dify-helm) chart (pinned in `manifests/staging/apps/dify/kustomization.yaml`) and managed by Argo CD.

## Prerequisites

- Cluster running: ingress-nginx, cert-manager, Argo CD
- `kubectl` configured (`kubeconfig.yaml` locally, never commit)
- Enough capacity on 2 worker nodes (Dify runs PostgreSQL, Redis, Weaviate, API, worker, web, sandbox, proxy, etc.)

## 0. Enable Helm for Kustomize in Argo CD (required once)

Dify uses Kustomize `helmCharts`. Argo CD must pass `--enable-helm` via the **`argocd-cm`** ConfigMap (not `argocd-cmd-params-cm`).

```bash
export KUBECONFIG=/Users/alireza/platform/vultr/kubeconfig.yaml

kubectl patch configmap argocd-cm -n argocd --type merge \
  --patch-file manifests/staging/argocd/argocd-cm-enable-helm.patch.yaml

kubectl -n argocd rollout restart deployment argocd-repo-server

# Confirm the setting is present:
kubectl -n argocd get configmap argocd-cm -o yaml | grep enable-helm
```

Wait for `argocd-repo-server` to be Ready, then continue. Without this step, the `dify` Application stays `Unknown` with `must specify --enable-helm`.

## 1. Set Dify secret key (required for production; recommended before first use)

Generate a secret in the cluster (not stored in Git):

```bash
kubectl create namespace dify --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic dify-global-secret -n dify \
  --from-literal=app-secret-key="$(openssl rand -base64 42)" \
  --dry-run=client -o yaml | kubectl apply -f -
```

`values.yaml` wires `SECRET_KEY` for `api` and `worker` from that Secret (`secretKeyRef` → `dify-global-secret` / `app-secret-key`). No secret value is stored in Git.

Create the Secret **before** API/worker pods start (or restart them after creating it):

```bash
kubectl -n dify rollout restart deployment -l app.kubernetes.io/instance=dify
```

## 2. Commit and push Git changes

```bash
git add manifests/staging/argocd-apps/dify.yaml \
        manifests/staging/apps/dify/ \
        docs/deploy-dify.md
git commit -m "Add Dify staging deployment via Argo CD"
git push origin main
```

Remove `fire` from the cluster if it was applied earlier:

```bash
kubectl delete application fire -n argocd --ignore-not-found
```

## 3. Apply the Argo CD Application (one-time bootstrap)

```bash
kubectl apply -f manifests/staging/argocd-apps/dify.yaml
```

Watch sync:

```bash
kubectl -n argocd get application dify -w
kubectl -n dify get pods -w
```

First sync can take several minutes (many images + PVC binding).

## 4. Verify ingress and TLS

```bash
kubectl -n dify get ingress
kubectl -n dify get certificate
```

Open: **https://dify.70.34.202.247.nip.io/install**  
Complete the admin account setup, then log in at **https://dify.70.34.202.247.nip.io**.

## 5. Argo CD notes

- Path `manifests/staging/apps/dify` uses Kustomize **helmCharts** (requires step 0: `kustomize.buildOptions: --enable-helm` on `argocd-cm`).
- Patch file: `manifests/staging/argocd/argocd-cm-enable-helm.patch.yaml` (merge patch only; do not `kubectl apply` it as a full ConfigMap).
- `prune: false` avoids deleting PVCs/data on accidental manifest changes; enable after you trust the manifest set.
- To change the public hostname, edit `values.yaml` (`global.*Domain` and `ingress.hosts`) and sync.

## 6. Fix failed PVCs after storage config change (Vultr)

If pods were **Pending** with `access mode is not supported`, set RWO under `api.persistence.persistentVolumeClaim` and `pluginDaemon.persistence.persistentVolumeClaim` (not top-level `persistence.accessModes` — the chart reads the nested key).

After pushing updated `values.yaml`, clean up stuck resources (no data to keep yet):

```bash
export KUBECONFIG=/Users/alireza/platform/vultr/kubeconfig.yaml

# Remove failed / old PVCs so they can be recreated
kubectl delete pvc -n dify --all

# Remove read replica left from earlier chart defaults (prune is off)
kubectl delete sts dify-postgresql-read -n dify --ignore-not-found
kubectl delete svc dify-postgresql-read dify-postgresql-read-hl -n dify --ignore-not-found

# Refresh Argo and sync
kubectl -n argocd annotate application dify argocd.argoproj.io/refresh=hard --overwrite

kubectl get pvc,pods -n dify -w
```

If a PVC stays `Pending`, check: `kubectl get storageclass` and align `global.storageClass` in `values.yaml` (e.g. `vultr-block-storage` vs `vultr-block-storage-hdd`).

## 7. Troubleshooting

| Issue | Check |
|-------|--------|
| Pending pods | `kubectl describe pod -n dify <name>` — often CPU/memory or PVC |
| `access mode is not supported` | RWX on Vultr block storage — use updated `values.yaml` (api/plugin persistence off) |
| Ingress 502 | Pods ready? `kubectl -n dify get pods` |
| Certificate not ready | `kubectl -n dify describe certificate dify-tls` |
| Argo sync failed on Helm | Run step 0; `describe application dify` should not show `must specify --enable-helm` |
| `must specify --enable-helm` | Patch `argocd-cm` and restart `argocd-repo-server` (step 0) |

## 8. After install

- **Tools → MCP** to attach MCP servers (HTTP transport).
- **API keys** per app under API Access in the Dify UI.
- Do not commit LLM provider API keys; configure in Dify Settings or via cluster Secrets.
