# Kubernetes GitOps Observability

## Alertmanager Local Secret
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
