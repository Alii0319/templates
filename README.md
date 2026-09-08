# ArgoCD Production Deployment Guide

This document is a copy-paste-ready guide to install and configure **ArgoCD** on a Kubernetes cluster and connect it to this repository for GitOps-based continuous deployment.

Every command below is self-contained — copy the block, replace any `<PLACEHOLDER>`, and run it.

---

## Prerequisites

- A running Kubernetes cluster with `kubectl` access configured (`kubectl cluster-info` should work)
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) installed locally
- A Git repository containing your Kubernetes manifests (or Helm chart / Kustomize overlay)
- Cluster-admin permissions (installing ArgoCD requires creating CRDs and cluster roles)

---

## 1. Create the ArgoCD namespace

ArgoCD's own components (server, repo-server, application-controller, redis) live in an isolated namespace.

```bash
kubectl create namespace argocd
```

---

## 2. Install ArgoCD

Choose **one** of the two options below.

**Standard install** — fine for a single-node/dev cluster:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**HA install (recommended for production)** — runs multiple replicas of each ArgoCD component plus Redis HA, so ArgoCD survives a node failure:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

---

## 3. Wait for all ArgoCD pods to be ready

Prevents race conditions in the next steps by blocking until every deployment reports `Available`.

```bash
kubectl wait --for=condition=available --timeout=300s deployment --all -n argocd
```

---

## 4. Expose the ArgoCD API/UI server

Pick the option that matches your infrastructure.

**Option A — Cloud provider with LoadBalancer support** (AWS/GCP/Azure/DigitalOcean):

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

```bash
kubectl get svc argocd-server -n argocd
```

**Option B — Bare metal / VPS (no cloud LoadBalancer) — production standard.**
Use an Ingress with TLS instead of LoadBalancer, since `LoadBalancer` type will stay `<pending>` forever without a cloud controller.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: <CERT_MANAGER_ISSUER>   # e.g. letsencrypt-prod
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - <ARGOCD_DOMAIN>          # e.g. argocd.yourdomain.com
      secretName: argocd-server-tls
  rules:
    - host: <ARGOCD_DOMAIN>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: https
```

**Option C — Local testing only (never for production):**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

## 5. Retrieve the initial admin password

ArgoCD auto-generates an admin password on first install, stored as a secret.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

> Username is `admin`. Log in immediately and change this password, then delete the secret (step 7).

---

## 6. Log in via ArgoCD CLI

```bash
argocd login <ARGOCD_SERVER_IP_OR_DOMAIN>
```

Change the password right after logging in:

```bash
argocd account update-password
```

---

## 7. Delete the initial admin secret

Once you've changed the password, the auto-generated secret is no longer needed and should be removed.

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## 8. Add your Git repository

Skip this step if your repository is public.

```bash
argocd repo add <GIT_REPO_URL> --username <GIT_USERNAME> --password <GITHUB_PAT>
```

> Use a GitHub Personal Access Token as the password, never your real account password.

---

## 9. Apply the Application manifest

Save the block below as `argocd-application.yaml`, fill in every `<PLACEHOLDER>`, then apply it.

```yaml
# ---------------------------------------------------------------------------
# ArgoCD Application manifest — production-standard GitOps template
# ---------------------------------------------------------------------------

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <PROJECT_NAME>              # e.g. ecommerce-scraper-engine
  namespace: argocd                 # Application resource always lives in argocd namespace
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # ensures cleanup on deletion
  labels:
    app: <PROJECT_NAME>
    env: <ENVIRONMENT>              # e.g. production / staging

spec:
  project: default                  # or a custom AppProject for RBAC scoping

  source:
    repoURL: <GIT_REPO_URL>         # e.g. https://github.com/Alii0319/<repo>.git
    targetRevision: <BRANCH>        # e.g. main
    path: <PATH_TO_MANIFESTS>       # e.g. k8s/ or deploy/overlays/production

    # --- Pick ONE based on how your manifests are structured ---

    # Option A: Plain Kubernetes YAML — no extra config needed.

    # Option B: Kustomize
    # kustomize:
    #   namePrefix: <PROJECT_NAME>-

    # Option C: Helm chart
    # helm:
    #   releaseName: <PROJECT_NAME>
    #   valueFiles:
    #     - values-production.yaml
    #   parameters:
    #     - name: image.tag
    #       value: <IMAGE_TAG>

  destination:
    server: https://kubernetes.default.svc   # same cluster ArgoCD runs on
    namespace: <TARGET_NAMESPACE>             # e.g. ecommerce-scraper-prod

  syncPolicy:
    automated:
      prune: true        # remove resources deleted from git
      selfHeal: true      # auto-correct manual cluster drift
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas   # remove if you don't use an HPA

---
# OPTIONAL: Custom AppProject — recommended once you manage multiple apps.
# Restricts what this Application is allowed to deploy.
# apiVersion: argoproj.io/v1alpha1
# kind: AppProject
# metadata:
#   name: <PROJECT_NAME>-project
#   namespace: argocd
# spec:
#   description: <PROJECT_NAME> project scope
#   sourceRepos:
#     - <GIT_REPO_URL>
#   destinations:
#     - namespace: <TARGET_NAMESPACE>
#       server: https://kubernetes.default.svc
#   clusterResourceWhitelist:
#     - group: ''
#       kind: Namespace
#   namespaceResourceWhitelist:
#     - group: '*'
#       kind: '*'
```

Apply it:

```bash
kubectl apply -f argocd-application.yaml
```

From now on, deploying a new version is just `git push` — ArgoCD detects the change and syncs automatically.

---

## 10. Verify the sync

```bash
argocd app get <PROJECT_NAME>
```

```bash
argocd app sync <PROJECT_NAME>
```

```bash
kubectl get pods -n <TARGET_NAMESPACE>
```

---

## Production Hardening Checklist

| Item | Why |
|---|---|
| Use HA install (step 2, option 2) | Survives node/pod failure |
| Ingress + TLS instead of `LoadBalancer` on bare metal | `LoadBalancer` never resolves without a cloud controller |
| Pin image tags to commit SHA or semver, never `latest` | Same tag = ArgoCD sees "already synced", won't redeploy |
| Set resource `requests`/`limits` in your app manifests | Prevents one app starving the cluster |
| Configure SSO (OIDC/SAML) and disable the local `admin` account after | Local admin account is a single point of compromise |
| Create a custom `AppProject` per app/team | Restricts source repos and destinations — real RBAC |
| Enable ArgoCD metrics + Prometheus/Grafana | Visibility into sync failures and drift |
| Set `automated.prune` + `selfHeal` deliberately | Any manual `kubectl edit` will be reverted — make sure your team knows this |

---

## Useful Commands Reference

```bash
# List all ArgoCD-managed applications
argocd app list

# View detailed sync/health status of one app
argocd app get <PROJECT_NAME>

# Force a manual sync
argocd app sync <PROJECT_NAME>

# Roll back to a previous deployed revision
argocd app rollback <PROJECT_NAME> <REVISION_ID>

# Delete an application (and optionally its resources)
argocd app delete <PROJECT_NAME> --cascade
```
