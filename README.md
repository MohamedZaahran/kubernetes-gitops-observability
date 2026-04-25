# Kubernetes GitOps Observability

Production-style Kubernetes GitOps observability lab built with ArgoCD, Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter.

The project demonstrates GitOps-based deployment, metrics collection, dashboard provisioning, alerting, email notifications, infrastructure monitoring, and failure simulation on Kubernetes.

---

## Overview

This project deploys a complete observability stack on Kubernetes using GitOps principles.

ArgoCD continuously reconciles the desired state from Git, while Prometheus collects metrics from the application, Kubernetes objects, monitoring stack components, and node-level exporters. Grafana visualizes the system state, and Alertmanager routes Prometheus alerts to Gmail.

The `podinfo` application is used as the demo workload for testing application monitoring, deployment health, alerting, and failure scenarios.

---

## Architecture

```text
GitHub Repository
      |
      v
ArgoCD App-of-Apps
      |
      v
Kubernetes Cluster
      |
      |-- podinfo
      |-- Prometheus
      |-- Grafana
      |-- Alertmanager
      |-- kube-state-metrics
      |-- node-exporter
```

Monitoring flow:

```text
podinfo / kube-state-metrics / node-exporter / Grafana / Alertmanager
        |
        v
Prometheus
        |
        |-- Grafana dashboards
        |
        v
Alertmanager
        |
        v
Gmail notifications
```

![ArgoCD App of Apps](screenshots/argocd-app-of-apps.jpg)

---

## Tech Stack

- Kubernetes
- Minikube
- ArgoCD
- Kustomize
- Prometheus
- Grafana
- Alertmanager
- kube-state-metrics
- node-exporter
- Gmail SMTP
- GitHub

---

## Features

- GitOps deployment using ArgoCD App-of-Apps
- ArgoCD automated sync, prune, and self-heal
- Prometheus annotation-based pod scraping
- Prometheus rule files managed through Git
- Grafana datasource and dashboard provisioning
- Alertmanager Gmail notifications
- Alert grouping, repeat intervals, severity-based routing, and resolved notifications
- kube-state-metrics for Kubernetes object-state metrics
- node-exporter for node CPU, memory, and disk metrics
- PVC-backed persistence for Prometheus and Grafana
- App-level, monitoring-stack, and node-level alerts
- Failure scenarios tested and documented
- Secret values kept out of public Git

---

## Repository Structure

```text
argocd/
monitoring/
podinfo/
failure-scenarios.md
README.md
```

### Main directories

- `argocd/`  
  ArgoCD application manifests and App-of-Apps configuration.

- `monitoring/`  
  Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter manifests.

- `podinfo/`  
  Demo application manifests used for monitoring and failure testing.

- `failure-scenarios.md`  
  Documented failure scenarios, expected symptoms, alerts, dashboard behavior, and recovery steps.

---

## Prerequisites

- WSL2 or Linux shell
- Docker Desktop or compatible container runtime
- Minikube
- kubectl
- Git
- Gmail account with an App Password for email alerts

Optional:
- ArgoCD CLI
- VS Code
- Codex or another coding assistant for editing and troubleshooting

---

## Installation / Setup

### 1. Clone the repository

```bash
git clone https://github.com/MohamedZaahran/kubernetes-gitops-observability.git
cd kubernetes-gitops-observability
```

### 2. Start Minikube

```bash
minikube start
```

### 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until ArgoCD pods are running:

```bash
kubectl get pods -n argocd
```

### 4. Create the Alertmanager Gmail Secret

Alertmanager Gmail credentials are not committed to Git.

Create the monitoring namespace first:

```bash
kubectl create namespace prometheus --dry-run=client -o yaml | kubectl apply -f -
```

Keep `monitoring/alertmanager/alertmanager-secret.example.yaml` committed with placeholder values only.

Create a local ignored secret file:

```bash
cp monitoring/alertmanager/alertmanager-secret.example.yaml monitoring/alertmanager/alertmanager-secret.yaml
```

Edit `monitoring/alertmanager/alertmanager-secret.yaml` and replace:

```yaml
auth_password: "WRITE_YOUR_PASSWORD_HERE"
```

with your Gmail App Password.

Create or update the Kubernetes Secret:

```bash
kubectl create secret generic alertmanager-config-secret -n prometheus --from-file=alertmanager.yml=monitoring/alertmanager/alertmanager-secret.yaml --dry-run=client -o yaml | kubectl apply -f -
```

The Secret created is:

```text
alertmanager-config-secret
```

with the key:

```text
alertmanager.yml
```

Do not commit `alertmanager-secret.yaml` or `alertmanager-secret.yml`.

### 5. Bootstrap the ArgoCD App-of-Apps

Apply the root ArgoCD application manifest:

```bash
kubectl apply -f root-app.yaml -n argocd
```

ArgoCD will then create the child applications and sync the monitoring stack and podinfo workload from Git.

### 6. Verify deployment

```bash
kubectl get applications -n argocd
kubectl get pods -A
kubectl get pvc -n prometheus
```

Expected result:
- ArgoCD apps are `Synced` and `Healthy`
- Prometheus is running
- Grafana is running
- Alertmanager is running
- kube-state-metrics is running
- node-exporter is running
- Prometheus and Grafana PVCs are `Bound`

---

## Accessing the Tools

### Grafana

```bash
kubectl port-forward svc/grafana-service -n prometheus 3000:3000
```

Open:

```text
http://localhost:3000
```

Login:

```text
Username: admin
Password: admin123
```

### Prometheus

```bash
kubectl port-forward svc/prometheus-service -n prometheus 9090:80
```

Open:

```text
http://localhost:9090
```

### Alertmanager

```bash
kubectl port-forward svc/alertmanager -n prometheus 9093:9093
```

Open:

```text
http://localhost:9093
```

### ArgoCD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Get the initial ArgoCD admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

---

## Monitoring and Alerting

Prometheus loads alert rules from:

```text
/etc/prometheus/rules/alert.rules.yml
```

The rules are grouped into:

- `pod-health`
- `node-health`

---

## Alerts Implemented

### Application alerts

| Alert | Purpose |
|---|---|
| `PodInfoScrapeDown` | Detects when podinfo exists but Prometheus cannot scrape it |
| `PodInfoMissingTargets` | Detects when podinfo scrape targets disappear |
| `PodInfoRestarting` | Detects podinfo container restarts |
| `PodinfoReplicasMismatch` | Detects desired replicas not matching available replicas |
| `PodinfoUnavailableReplicas` | Detects unavailable podinfo replicas |

### Monitoring stack alerts

| Alert | Purpose |
|---|---|
| `PrometheusDown` | Detects when Prometheus deployment has no available replicas |
| `GrafanaTargetDown` | Detects Grafana scrape failure |
| `GrafanaMissingTargets` | Detects missing Grafana scrape targets |
| `AlertmanagerTargetDown` | Detects Alertmanager scrape failure |
| `AlertmanagerMissingTargets` | Detects missing Alertmanager scrape targets |
| `KubeStateMetricsTargetDown` | Detects kube-state-metrics scrape failure |
| `KubeStateMetricsMissingTargets` | Detects missing kube-state-metrics scrape targets |
| `NodeExporterTargetDown` | Detects node-exporter scrape failure |
| `NodeExporterMissingTargets` | Detects unavailable node-exporter DaemonSet targets |

### Node health alerts

| Alert | Threshold |
|---|---|
| `HighCPUUsage` | CPU usage above 80% for more than 2 minutes |
| `LowMemoryAvailable` | Available memory below 20% for more than 2 minutes |
| `HighDiskUsage` | Disk usage on `/var` above 80% |

### Important alerting concepts

```text
up == 0
```

means the target exists, but Prometheus failed to scrape it.

```text
absent(up{...})
```

means the target disappeared entirely.

For kube-state-metrics workload metrics:
- `namespace` / `pod` describe the exporter pod
- `exported_namespace` / `exported_pod` describe the actual Kubernetes object

Example:

```promql
kube_pod_container_status_restarts_total{exported_namespace="podinfo", container="podinfo"}
```

---

## Alertmanager

Alertmanager is configured to:
- receive alerts from Prometheus
- group alerts by `alertname` and `severity`
- send firing and resolved notifications
- route warning and critical alerts to Gmail

Example routing behavior:

```text
Prometheus alert -> Alertmanager -> Gmail notification
```

![Alertmanager Alerts](screenshots/alertmanager-alerts.png)

### Gmail notifications

![Gmail High CPU Firing](screenshots/gmail-high-cpu-firing.jpg)

![Gmail High CPU Resolved](screenshots/gmail-high-cpu-resolved.jpg)

---

## Grafana Dashboard

The Grafana dashboard visualizes application, monitoring-stack, and node-level health.

Panels include:
- Podinfo traffic rate
- Desired replicas
- Available replicas
- Unavailable replicas
- Container restarts
- Grafana UP
- Alertmanager UP
- kube-state-metrics UP
- node-exporter UP
- node CPU usage
- node memory usage
- node disk usage trend
- node disk usage gauge

![Grafana Dashboard](screenshots/grafana-dashboard.png)

### Dashboard design notes

- Stat panels are used for current-value signals.
- Time series panels are used for trends.
- UP/DOWN panels map:
  - `1` -> `UP`
  - `0` -> `DOWN`
- Dashboard default time range is intended for recent troubleshooting, usually around `now-1h`.

Example UP query:

```promql
max(up{job="kubernetes-pods", namespace="prometheus", pod=~"grafana-.*"}) OR on() vector(0)
```

---

## Prometheus Alerts Page

![Prometheus Alerts](screenshots/prometheus-alerts.png)

---

## Failure Scenarios Tested

Detailed scenarios are documented in:

[failure-scenarios.md](failure-scenarios.md)

Examples tested:
- ArgoCD self-healing after manual deployment image change
- ArgoCD recovery after namespace deletion
- podinfo scrape failure using wrong metrics port
- missing podinfo scrape targets
- podinfo container restart detection
- bad image tag rollout failure
- Grafana outage
- Alertmanager outage
- kube-state-metrics outage
- Prometheus self-monitoring limitation
- Grafana PVC reset
- Alertmanager configuration failure
- node-exporter scrape failure
- high CPU usage
- low memory availability
- high disk usage

---

## Persistence

Prometheus and Grafana use PVC-backed storage.

### Prometheus

Prometheus data is stored under:

```text
/prometheus
```

### Grafana

Grafana data is stored under:

```text
/var/lib/grafana
```

Important behavior:
- deleting a pod does not delete persisted data
- deleting the PVC resets stored state
- provisioned dashboards come from Git/config files
- UI-created Grafana dashboards are stored in Grafana's database on the PVC

---

## Secret Management

Alertmanager Gmail credentials are not stored in Git.

### Alertmanager Local Secret
1. Keep `monitoring/alertmanager/alertmanager-secret.example.yaml` committed with placeholder values only.
2. Create a local ignored secret file:
```bash
cp monitoring/alertmanager/alertmanager-secret.example.yaml monitoring/alertmanager/alertmanager-secret.yaml
```
3. Edit `monitoring/alertmanager/alertmanager-secret.yaml` and replace `WRITE_YOUR_PASSWORD_HERE` with the real Gmail app password.
4. Create or update the Kubernetes Secret:
```bash
kubectl create secret generic alertmanager-config-secret -n prometheus --from-file=alertmanager.yml=monitoring/alertmanager/alertmanager-secret.yaml --dry-run=client -o yaml | kubectl apply -f -
```

Do not commit `alertmanager-secret.yaml` or `alertmanager-secret.yml`.

For production-grade GitOps secret management, better options include:
- Sealed Secrets
- SOPS
- External Secrets Operator
- Vault

---

## Security Notes

- Do not commit real Gmail App Passwords to Git.
- Use Gmail App Passwords instead of the normal account password.
- Rotate credentials immediately if they are exposed.
- Kubernetes Secrets are acceptable for this lab, but they are not a complete production secret-management solution on their own.

---

## Lessons Learned

- Git is the source of truth in GitOps.
- ArgoCD restores live-cluster drift.
- If Git contains a broken desired state, ArgoCD enforces the broken state.
- `up == 0` and `absent(up{...})` represent different failure modes.
- kube-state-metrics shows Kubernetes object state.
- Direct scraping shows runtime endpoint health.
- Prometheus cannot fully monitor its own complete outage from inside itself.
- Alertmanager failure limits external notification delivery.
- PVC-backed applications keep state after pod deletion.
- Dashboards should explain alerts, not just display metrics.
- Secrets must not be committed to public repositories.

---

## Future Improvements

Possible future enhancements:
- Sealed Secrets, SOPS, or External Secrets Operator for GitOps-safe secrets
- Loki and Promtail for log aggregation
- Blackbox Exporter for external HTTP probing
- Ingress with TLS
- Multi-environment Kustomize overlays
- External Prometheus or uptime monitoring for Prometheus self-monitoring limitations
- More recording rules for simplified alert and dashboard queries

---

## Cleanup

To remove the deployed workloads:

```bash
kubectl delete -f root-app.yaml -n argocd
```

To remove namespaces and all related resources:

```bash
kubectl delete namespace prometheus podinfo argocd
```

Warning: this deletes the monitoring stack, application, PVCs, and ArgoCD resources.

---

## Author

**Mohamed Zahran**

- GitHub: [MohamedZaahran](https://github.com/MohamedZaahran)
- Project: [kubernetes-gitops-observability](https://github.com/MohamedZaahran/kubernetes-gitops-observability)
