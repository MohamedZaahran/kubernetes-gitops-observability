# Failure Scenarios

## Purpose

This document records the main failure scenarios tested in the Kubernetes GitOps Observability project.

Each scenario includes:
- what was broken
- why it matters
- expected symptoms
- expected alerts
- expected Grafana behavior
- recovery steps
- key lesson learned

---

## 1. ArgoCD Self-Healing: Manual Deployment Image Change

### What was broken
A live Kubernetes Deployment image was manually changed using `kubectl`.

### Why this matters
This tests whether ArgoCD detects live-cluster drift and restores the desired state from Git.

### How to simulate
```bash
kubectl set image deployment/<deployment-name> <container-name>=<wrong-image> -n <namespace>
```

### Expected symptoms
- Live Deployment changes temporarily
- ArgoCD detects drift
- ArgoCD restores the image from Git

### Expected alerts
Usually no alert should fire if ArgoCD fixes the drift quickly.

### Expected Grafana behavior
Usually no visible long-term impact.

### Recovery
No manual recovery should be needed if ArgoCD self-heal is enabled.

### Lesson learned
Manual changes to GitOps-managed resources are temporary. Git remains the source of truth.

---

## 2. ArgoCD Self-Healing: Namespace Deletion

### What was broken
A GitOps-managed namespace was deleted manually.

### Why this matters
This tests whether ArgoCD can recreate deleted resources and restore the declared state.

### How to simulate
```bash
kubectl delete namespace podinfo
```

### Expected symptoms
- Namespace disappears temporarily
- Workloads disappear temporarily
- ArgoCD recreates the namespace and resources

### Expected alerts
Depending on timing:
- `PodInfoMissingTargets`
- `PodInfoScrapeDown`
- deployment-related podinfo alerts may fire if outage is long enough

### Expected Grafana behavior
- Podinfo UP may go DOWN or show missing data
- Traffic may drop
- Available replicas may drop temporarily

### Recovery
ArgoCD should recreate the namespace and resources automatically.

### Lesson learned
ArgoCD can restore Kubernetes objects, but only if they are declared in Git.

---

## 3. Podinfo Scrape Failure: Wrong Metrics Port

### What was broken
The podinfo Prometheus scrape port annotation was changed to an incorrect port.

### Why this matters
This tests the difference between a target that exists but cannot be scraped and a target that disappears entirely.

### How to simulate
Change:

```yaml
prometheus.io/port: "9898"
```

to:

```yaml
prometheus.io/port: "9999"
```

Then sync through ArgoCD.

### Expected symptoms
- Prometheus still discovers the podinfo target
- Scrape fails
- `up` becomes `0`

### Expected alerts
- `PodInfoScrapeDown`

### Expected Grafana behavior
- Podinfo UP panel shows DOWN
- Traffic panel may drop or show broken scraping
- Alert appears in Prometheus and Alertmanager
- Email notification should be sent

### Recovery
Restore the correct annotation:

```yaml
prometheus.io/port: "9898"
```

Then sync through ArgoCD.

### Lesson learned
`up == 0` means the target exists, but Prometheus failed to scrape it.

---

## 4. Podinfo Missing Target: Scrape Annotation Removed

### What was broken
The podinfo scrape annotation was removed or changed so Prometheus no longer discovers the target.

### Why this matters
This tests the difference between a broken target and a missing target.

### How to simulate
Change or remove:

```yaml
prometheus.io/scrape: "true"
```

For example:

```yaml
prometheus.io/scrape: "false"
```

### Expected symptoms
- Podinfo target disappears from Prometheus target list
- No `up` series exists for podinfo

### Expected alerts
- `PodInfoMissingTargets`

### Expected Grafana behavior
- Podinfo UP panel shows DOWN if using:
```promql
max(up{job="kubernetes-pods", namespace="podinfo"}) OR on() vector(0)
```
- Traffic may disappear or show no data

### Recovery
Restore:

```yaml
prometheus.io/scrape: "true"
```

Then sync through ArgoCD.

### Lesson learned
`absent(up{...})` is needed when a target disappears entirely.

---

## 5. Podinfo Restart Alert

### What was broken
The podinfo container was made to restart.

### Why this matters
This tests Kubernetes restart detection using kube-state-metrics.

### Query used
```promql
increase(kube_pod_container_status_restarts_total{exported_namespace="podinfo", container="podinfo"}[5m]) > 0
```

### Expected symptoms
- Restart counter increases
- kube-state-metrics exposes the updated restart count

### Expected alerts
- `PodInfoRestarting`

### Expected Grafana behavior
- Container Restarts panel shows an increase

### Recovery
Fix the cause of the restart.

### Lesson learned
For kube-state-metrics workload metrics, use `exported_namespace` and `exported_pod`, not `namespace` and `pod`.

---

## 6. Bad Podinfo Image Tag

### What was broken
The podinfo image was changed to a missing or invalid image tag.

### Why this matters
This simulates a real broken rollout.

### How to simulate
Change the podinfo image in Git to an invalid tag, then let ArgoCD sync.

### Expected symptoms
- New pods fail to start
- Pods may show:
  - `ErrImagePull`
  - `ImagePullBackOff`
- Desired replicas remain unchanged
- Available replicas drop
- Unavailable replicas increase

### Expected alerts
- `PodinfoReplicasMismatch`
- `PodinfoUnavailableReplicas`
- Possibly `PodInfoMissingTargets`
- Possibly `PodInfoScrapeDown`

### Expected Grafana behavior
- Desired Replicas remains stable
- Available Replicas decreases
- Unavailable Replicas increases
- Podinfo traffic may drop
- Podinfo UP may go DOWN or missing

### Recovery
Restore the correct image tag in Git and sync with ArgoCD.

### Useful commands
```bash
kubectl get pods -n podinfo
kubectl describe pod <pod-name> -n podinfo
```

### Lesson learned
Bad image rollouts are better alert tests than deleting pods because the failure persists long enough to observe.

---

## 7. Grafana Down: Scale Deployment to Zero

### What was broken
Grafana was scaled to zero replicas.

### Why this matters
This tests monitoring stack component availability and notification delivery.

### How to simulate
```bash
kubectl scale deployment grafana -n prometheus --replicas=0
```

For a GitOps-style test, make the change in Git and let ArgoCD sync.

### Expected symptoms
- Grafana pod disappears
- Grafana target disappears from Prometheus

### Expected alerts
- `GrafanaMissingTargets`

If Grafana exists but the scrape endpoint fails:
- `GrafanaTargetDown`

### Expected Grafana behavior
Grafana UI becomes unavailable, so dashboard cannot be used during the outage.

### Expected Alertmanager behavior
Email should still be sent because Prometheus and Alertmanager are still running.

### Recovery
Restore Grafana replicas to `1` in Git or by reverting the test change.

### Lesson learned
Grafana being down does not stop Prometheus or Alertmanager from detecting and notifying.

---

## 8. Alertmanager Down: Scale Deployment to Zero

### What was broken
Alertmanager was scaled to zero replicas.

### Why this matters
This tests the limitation of alert delivery when the notification component itself is down.

### How to simulate
```bash
kubectl scale deployment alertmanager -n prometheus --replicas=0
```

For a GitOps-style test, make the change in Git and let ArgoCD sync.

### Expected symptoms
- Alertmanager pod disappears
- Alertmanager target disappears from Prometheus

### Expected alerts
- `AlertmanagerMissingTargets`

If the target exists but scraping fails:
- `AlertmanagerTargetDown`

### Expected Grafana behavior
- Alertmanager UP panel shows DOWN

### Expected email behavior
Email may not arrive because Alertmanager is the component responsible for sending email.

### Recovery
Restore Alertmanager replicas to `1`.

### Lesson learned
Prometheus can detect Alertmanager failure, but external notification is limited if Alertmanager itself is down.

---

## 9. kube-state-metrics Down

### What was broken
kube-state-metrics was scaled to zero replicas.

### Why this matters
This tests whether the monitoring system detects loss of Kubernetes object-state metrics.

### How to simulate
```bash
kubectl scale deployment kube-state-metrics -n prometheus --replicas=0
```

### Expected symptoms
- kube-state-metrics target disappears
- Kubernetes object-state metrics stop updating
- Deployment-based alerts become unreliable

### Expected alerts
- `KubeStateMetricsMissingTargets`

If the target exists but scraping fails:
- `KubeStateMetricsTargetDown`

### Expected Grafana behavior
- Kube-State-Metrics UP panel shows DOWN
- Panels based on kube-state-metrics may show stale or missing data

### Recovery
Restore kube-state-metrics replicas to `1`.

### Lesson learned
kube-state-metrics is a dependency for Kubernetes-state alerts. If it is down, deployment and pod-state alerts become blind or stale.

---

## 10. Prometheus Down

### What was broken
Prometheus was scaled to zero replicas.

### Why this matters
This demonstrates a core limitation of self-monitoring.

### How to simulate
```bash
kubectl scale deployment prometheus-server -n prometheus --replicas=0
```

### Expected symptoms
- Prometheus stops evaluating rules
- Prometheus stops sending alerts to Alertmanager
- Grafana panels depending on Prometheus stop updating

### Expected alerts
`PrometheusDown` may not externally notify because Prometheus is required to evaluate and send alerts.

### Expected Alertmanager behavior
Existing alerts may eventually disappear or resolve because Prometheus stops refreshing them.

### Recovery
Restore Prometheus replicas to `1`.

### Lesson learned
Prometheus cannot reliably alert about its own complete failure from inside itself. Production systems usually use external monitoring or another Prometheus instance.

---

## 11. Alertmanager Config Error

### What was broken
Alertmanager configuration had invalid matcher syntax.

### Example broken config
```yaml
matchers:
  - severity: "warning"
```

### Correct config
```yaml
matchers:
  - severity="warning"
```

### Expected symptoms
- Alertmanager pod enters `CrashLoopBackOff`
- Logs show YAML unmarshal errors

### Useful command
```bash
kubectl logs deployment/alertmanager -n prometheus
```

### Expected alerts
If Prometheus is scraping Alertmanager:
- `AlertmanagerTargetDown`
- or `AlertmanagerMissingTargets`

### Recovery
Fix the Alertmanager config, recreate/update the Secret, and restart Alertmanager.

### Lesson learned
Alertmanager `matchers` expect strings, not YAML maps.

---

## 12. Grafana PVC Reset

### What was broken
Grafana PVC was deleted to reset Grafana state.

### Why this matters
This tests the difference between Git-provisioned dashboards and UI-created dashboards.

### How to simulate
First stop Grafana from using the PVC, then delete it.

```bash
kubectl scale deployment grafana -n prometheus --replicas=0
kubectl delete pvc grafana-pvc -n prometheus
```

### Expected symptoms
- Grafana internal DB is reset
- UI-created dashboards disappear
- provisioned dashboards return from Git/config files

### Possible issue
PVC may get stuck in `Terminating`.

### Cause
The PVC is still being used by a running Grafana pod.

### Recovery
Ensure no Grafana pod is using the PVC, then delete again.

### Lesson learned
Deleting a Deployment does not reset Grafana state if the PVC still exists. Deleting the PVC resets Grafana DB state.

---

## 13. Node Exporter Missing Target

### What was broken
Node Exporter was stopped or removed.

### Why this matters
This validates infrastructure/node monitoring.

### How to simulate
Since Node Exporter runs as a DaemonSet, delete the pod or temporarily change the DaemonSet through Git.

Example live-cluster test:
```bash
kubectl delete pod -l app=node-exporter -n prometheus
```

If the DaemonSet recreates it too quickly, use a Git-based test by removing/changing the DaemonSet or its scrape annotations.

### Expected symptoms
- Node Exporter target disappears or becomes unavailable
- Node-level metrics stop updating

### Expected alerts
- `NodeExporterMissingTargets`

If target exists but scrape fails:
- `NodeExporterTargetDown`

### Expected Grafana behavior
- Node-Exporter Up panel shows DOWN
- Node CPU/memory/disk panels may show missing or stale data

### Recovery
Restore Node Exporter DaemonSet and scrape configuration.

### Lesson learned
Node Exporter is the infrastructure metrics source. Without it, node-level observability is lost.

---

## 14. Node Exporter Scrape Failure

### What was broken
Node Exporter target exists but Prometheus cannot scrape it.

### Why this matters
This tests runtime scrape failure for infrastructure monitoring.

### How to simulate
Break the scrape port/path in the Node Exporter scrape configuration or annotations.

### Expected symptoms
- Node Exporter target still exists
- Prometheus scrape fails
- `up` becomes `0`

### Expected alerts
- `NodeExporterTargetDown`

### Expected Grafana behavior
- Node-Exporter Up panel shows DOWN
- Node metrics may stop updating

### Recovery
Restore correct Node Exporter scrape configuration.

### Lesson learned
TargetDown and MissingTargets are different failure modes and both should be tested.

---

## 15. High Node CPU Usage

### What was broken
CPU load was intentionally increased on the node.

### Why this matters
This tests node-level resource alerts.

### How to simulate
Run a CPU stress workload.

Example:
```bash
kubectl run cpu-stress -n prometheus --image=busybox --restart=Never -- sh -c "while true; do :; done"
```

Depending on cluster resources, more replicas or a stress image may be needed.

### Expected symptoms
- Node CPU usage increases
- CPU trend panel rises

### Expected alerts
- High CPU alert may fire if threshold and duration are reached

### Expected Grafana behavior
- Node CPU usage panel increases

### Recovery
```bash
kubectl delete pod cpu-stress -n prometheus
```

### Lesson learned
Resource alerts must be tested with realistic thresholds and enough sustained load.

---

## 16. Low Node Memory Available

### What was broken
Memory pressure was simulated.

### Why this matters
This tests node memory alerting.

### How to simulate
Run a memory-consuming workload.

Example:
```bash
kubectl run memory-stress -n prometheus --image=polinux/stress --restart=Never -- stress --vm 1 --vm-bytes 256M --vm-hang 1
```

Adjust memory size depending on Minikube capacity.

### Expected symptoms
- Available memory decreases
- Memory panel changes

### Expected alerts
- Low memory alert may fire if threshold and duration are reached

### Expected Grafana behavior
- Node memory panel shows increased pressure

### Recovery
```bash
kubectl delete pod memory-stress -n prometheus
```

### Lesson learned
Memory alerts must be tuned carefully in small local clusters to avoid false positives or unstable tests.

---

## 17. High Disk Usage

### What was broken
Disk usage was increased.

### Why this matters
This tests disk usage alerting and Grafana disk panels.

### How to simulate
Create a temporary file inside a pod or on the node.

Example pod-based test:
```bash
kubectl run disk-fill -n prometheus --image=busybox --restart=Never -- sh -c "dd if=/dev/zero of=/tmp/bigfile bs=10M count=100 && sleep 3600"
```

This may not affect the node filesystem enough depending on runtime storage behavior.

### Expected symptoms
- Disk usage panel may increase
- Disk gauge may increase

### Expected alerts
- High disk usage alert may fire if threshold is reached

### Recovery
```bash
kubectl delete pod disk-fill -n prometheus
```

### Lesson learned
Disk tests depend heavily on the filesystem and mount being monitored. Not every pod-level write will affect the same filesystem used in the alert query.

---

## Summary of Key Lessons

- Git is the source of truth in GitOps.
- ArgoCD fixes live drift, but enforces broken Git if Git is wrong.
- `up == 0` means target exists but scrape failed.
- `absent(up{...})` means target disappeared.
- kube-state-metrics shows Kubernetes object state.
- Direct scraping shows runtime endpoint health.
- Prometheus cannot reliably notify about its own full outage.
- Alertmanager failure limits external notification.
- PVC-backed apps keep state even after Deployment deletion.
- Dashboards should explain alerts, not just show graphs.
- Failure scenarios must be tested and documented to prove the system works.
