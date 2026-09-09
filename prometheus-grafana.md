# Prometheus + Grafana Observability Guide

Copy-paste-ready setup for cluster and application monitoring using **kube-prometheus-stack** — the industry-standard Helm chart bundling Prometheus, Grafana, Alertmanager, Prometheus Operator, node-exporter, and kube-state-metrics.

Replace every `<PLACEHOLDER>` before running a command.

---

## 0. Prerequisites

```bash
helm version
kubectl version --client
```

If Helm is missing:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

---

## 1. Add the Prometheus Community Helm repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## 2. Create the values file

Save as `values.yaml`:

```yaml
prometheus:
  prometheusSpec:
    retention: <RETENTION_PERIOD>          # e.g. 15d
    resources:
      requests:
        cpu: <PROM_CPU_REQUEST>            # e.g. 250m
        memory: <PROM_MEM_REQUEST>         # e.g. 512Mi
      limits:
        cpu: <PROM_CPU_LIMIT>              # e.g. 1
        memory: <PROM_MEM_LIMIT>           # e.g. 2Gi
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: <STORAGE_CLASS>   # e.g. standard, gp2, do-block-storage
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: <PROM_STORAGE_SIZE>    # e.g. 20Gi
    podMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false

grafana:
  persistence:
    enabled: true
    storageClassName: <STORAGE_CLASS>
    size: <GRAFANA_STORAGE_SIZE>            # e.g. 5Gi
  resources:
    requests:
      cpu: <GRAFANA_CPU_REQUEST>            # e.g. 100m
      memory: <GRAFANA_MEM_REQUEST>         # e.g. 256Mi
    limits:
      cpu: <GRAFANA_CPU_LIMIT>              # e.g. 500m
      memory: <GRAFANA_MEM_LIMIT>           # e.g. 512Mi
  # Leave adminPassword unset in production — the chart auto-generates a Secret.
  # For quick local/dev use, you may set: adminPassword: <YOUR_LOCAL_PASSWORD>

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: <STORAGE_CLASS>
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: <ALERTMANAGER_STORAGE_SIZE>   # e.g. 5Gi
```

---

## 3. Install the stack

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f values.yaml
```

---

## 4. Verify everything is running

```bash
kubectl wait --for=condition=ready pod --all -n monitoring --timeout=300s
kubectl get pods -n monitoring
```

---

## 5. Get the Grafana admin password

If you left `adminPassword` unset in `values.yaml` (recommended):

```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Username: `admin`. If you set `adminPassword` yourself, use that value instead.

---

## 6. Access the Grafana UI — pick one option

### Option A: Port-forward (fastest, no extra setup)

```bash
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80 --address=0.0.0.0 &
```

Open `http://localhost:3000` (or `http://<your-machine-IP>:3000` from another device on the network).

### Option B: LoadBalancer

```bash
kubectl patch svc kube-prometheus-stack-grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc kube-prometheus-stack-grafana -n monitoring
```

Open `http://<EXTERNAL-IP>`.

> On Minikube, `EXTERNAL-IP` stays `<pending>` unless you run `minikube tunnel` in a separate terminal first.

### Option C: Ingress with a local domain (closest to production, no port-forward needed)

Enable the ingress controller (Minikube):

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

Map the domain to your cluster IP:

```bash
echo "$(minikube ip) grafana.local" | sudo tee -a /etc/hosts
```

Save as `grafana-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "false"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-stack-grafana
                port:
                  number: 80
```

```bash
kubectl apply -f grafana-ingress.yaml
kubectl get ingress -n monitoring
```

Open `http://grafana.local` directly — no port-forward or background job needed.

> Grafana serves plain HTTP by default, so unlike Argo CD it needs no extra `insecure mode` ConfigMap patch to work behind a non-TLS Ingress.

---

## 7. Access the Prometheus UI (internal use only — never expose publicly)

```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090
```

Open `http://localhost:9090/targets` to confirm all targets are being scraped.

---

## 8. Scrape metrics from your own application

Add a `/metrics` endpoint to your app (e.g. `django-prometheus` for Django), then apply a `ServiceMonitor`.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: <PROJECT_NAME>-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: <PROJECT_NAME>              # must match your app's Service labels
  namespaceSelector:
    matchNames:
      - <TARGET_NAMESPACE>             # namespace where your app runs
  endpoints:
    - port: <METRICS_PORT_NAME>        # e.g. http, must match your Service's port name
      path: /metrics
      interval: 30s
```

```bash
kubectl apply -f servicemonitor.yaml
```

---

## 9. Basic alert rule example

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: <PROJECT_NAME>-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: <PROJECT_NAME>.rules
      rules:
        - alert: <PROJECT_NAME>HighErrorRate
          expr: rate(http_requests_total{job="<PROJECT_NAME>", status=~"5.."}[5m]) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on <PROJECT_NAME>"
            description: "More than 5% of requests are failing for the last 5 minutes."
```

```bash
kubectl apply -f prometheusrule.yaml
```

---

## 10. Upgrade or uninstall

```bash
# Upgrade after changing values.yaml
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f values.yaml

# Uninstall (does NOT remove CRDs)
helm uninstall kube-prometheus-stack -n monitoring
```

CRDs are not removed automatically — clean up manually if fully decommissioning:

```bash
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com \
  alertmanagers.monitoring.coreos.com \
  podmonitors.monitoring.coreos.com \
  probes.monitoring.coreos.com \
  prometheuses.monitoring.coreos.com \
  prometheusrules.monitoring.coreos.com \
  servicemonitors.monitoring.coreos.com \
  thanosrulers.monitoring.coreos.com
```

---

## Production Hardening Checklist

| Item | Why |
|---|---|
| Set `prometheusSpec.retention` explicitly | Default retention can silently fill disk over time |
| Use `storageSpec` with a real `storageClassName` | Without it, Prometheus data is lost on pod restart |
| Never hardcode Grafana admin password in `values.yaml` | Committed passwords in Git are a leak — let the chart auto-generate the Secret |
| Set resource `requests`/`limits` on Prometheus and Grafana | Prometheus is memory-hungry and can OOM-kill other pods without limits |
| Use Ingress + real TLS (cert-manager) instead of `.local` + plain HTTP | `.local` domains and `ssl-redirect: false` are for local dev only |
| Configure Alertmanager receivers (Slack/email/PagerDuty) | An alerting stack with no notification target is just a dashboard |
| Enable Grafana SSO and disable local admin login once configured | Local admin account is a single point of compromise |
| Set `serviceMonitorSelectorNilUsesHelmValues: false` | Otherwise Prometheus ignores ServiceMonitors outside the release's own namespace |

---

## Quick Reference

```bash
helm list -n monitoring                                # show installed release
helm get values kube-prometheus-stack -n monitoring     # see currently applied values
kubectl get servicemonitors -n monitoring               # list all scrape targets
kubectl get prometheusrules -n monitoring                # list all alert rules
kubectl get ingress -n monitoring                        # check ingress + assigned address
```
