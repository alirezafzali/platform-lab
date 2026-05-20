# Deploy Dify on staging (VKE + Argo CD)

Dify is deployed via the [dify-helm](https://github.com/BorisPolonsky/dify-helm) chart (pinned in `manifests/staging/apps/dify/kustomization.yaml`) and managed by Argo CD.

## Prerequisites

- Cluster running: ingress-nginx, cert-manager, Argo CD
- `kubectl` configured (`kubeconfig.yaml` locally, never commit)
- Enough capacity on 2 worker nodes (Dify runs PostgreSQL, Redis, Weaviate, API, worker, web, sandbox, proxy, etc.)

## 1. Set Dify secret key (required for production; recommended before first use)

Generate a secret in the cluster (not stored in Git):

```bash
kubectl create namespace dify --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic dify-global-secret -n dify \
  --from-literal=app-secret-key="$(openssl rand -base64 42)" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Then set `global.appSecretKey` in `manifests/staging/apps/dify/values.yaml` via one of:

- **Option A:** Copy the key into values (only for a private lab repo), or
- **Option B:** Use Argo CD UI → Application `dify` → Helm → override `global.appSecretKey`, or
- **Option C:** After first sync, `helm upgrade` with `--set global.appSecretKey=...` (if managing outside Argo)

For the first bootstrap, the chart default may apply if `appSecretKey` is empty; rotate before exposing publicly.

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

- Path `manifests/staging/apps/dify` uses Kustomize **helmCharts** (requires Argo CD with Helm enabled on the repo-server, default in recent versions).
- `prune: false` avoids deleting PVCs/data on accidental manifest changes; enable after you trust the manifest set.
- To change the public hostname, edit `values.yaml` (`global.*Domain` and `ingress.hosts`) and sync.

## 6. Troubleshooting

| Issue | Check |
|-------|--------|
| Pending pods | `kubectl describe pod -n dify <name>` — often CPU/memory or PVC |
| Ingress 502 | Pods ready? `kubectl -n dify get pods` |
| Certificate not ready | `kubectl -n dify describe certificate dify-tls` |
| Argo sync failed on Helm | Argo CD logs; ensure chart repo is reachable from cluster |

## 7. After install

- **Tools → MCP** to attach MCP servers (HTTP transport).
- **API keys** per app under API Access in the Dify UI.
- Do not commit LLM provider API keys; configure in Dify Settings or via cluster Secrets.
