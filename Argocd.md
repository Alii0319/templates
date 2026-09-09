# ArgoCD Deployment Guide

Copy-paste-ready setup for ArgoCD on Kubernetes with GitOps deployment via this repository.

Replace every `<PLACEHOLDER>` before running a command.

---

## 0. Install the ArgoCD CLI

**Linux:**

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

**macOS:**

```bash
brew install argocd
```

Verify:

```bash
argocd version --client
```

---

## 1. Create the ArgoCD namespace

```bash
kubectl create namespace argocd
```

---

## 2. Install ArgoCD

```bash
kubectl apply --server-side --force-conflicts -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml```

---

## 3. Wait for all ArgoCD pods to be ready

```bash
kubectl wait --for=condition=available --timeout=300s deployment --all -n argocd
```

---

## 4. Expose the ArgoCD server

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

---

## 5. Get the external IP and open the ArgoCD UI

```bash
kubectl get svc argocd-server -n argocd
```

Copy the value under `EXTERNAL-IP`, then open in browser:

```
https://<EXTERNAL-IP>
```

> If `EXTERNAL-IP` shows `<pending>`, your cluster has no cloud LoadBalancer support. Use this instead to access the UI locally:
> ```bash
> kubectl port-forward svc/argocd-server -n argocd 8080:443
> ```
> Then open `https://localhost:8080`

---

## 6. Retrieve the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Username: `admin`. Use this with the password above to log into the UI from step 5.

---

## 7. Log in via ArgoCD CLI and change password

```bash
argocd login <EXTERNAL-IP> --insecure
argocd account update-password
```

> `--insecure` is required because ArgoCD's default TLS cert is self-signed. Once you set up a real cert (via Ingress + cert-manager, or a company-issued cert), drop this flag.

---

## 8. Delete the initial admin secret

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## 9. Add your Git repository (skip if public)

```bash
argocd repo add <GIT_REPO_URL> --username <GIT_USERNAME> --password <GITHUB_PAT>
```

---

## 10. Apply the Application manifest

Save as `argocd-application.yaml`, fill in the placeholders, then apply:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <PROJECT_NAME>
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: <GIT_REPO_URL>
    targetRevision: <BRANCH>          # e.g., main ya master
    path: <PATH_TO_MANIFESTS_OR_HELM> # e.g., K8S/Umbrella-chart
    # Helm charts ke liye niche wale section ko uncomment karen:
    # helm:
    #   valueFiles:
    #     - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: <TARGET_NAMESPACE>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

```bash
kubectl apply -f argocd-application.yaml
```

From now on, deploying a new version is just `git push` — ArgoCD detects the change and syncs automatically.

---

## 11. Verify the sync

```bash
argocd app get <PROJECT_NAME>
kubectl get pods -n <TARGET_NAMESPACE>
```

---

## Quick Reference

```bash
argocd app list                              # list all apps
argocd app get <PROJECT_NAME>                 # status of one app
argocd app sync <PROJECT_NAME>                # force manual sync
argocd app rollback <PROJECT_NAME> <REV_ID>   # rollback
```
