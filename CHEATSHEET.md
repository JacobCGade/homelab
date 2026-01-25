# Homelab Operations Cheat Sheet

Quick reference for common operations in your K3s homelab cluster.

## Table of Contents
- [Cluster Access](#cluster-access)
- [GitOps - Adding/Updating Apps](#gitops---addingupdating-apps)
- [Certificate Management](#certificate-management)
- [Storage Operations](#storage-operations)
- [Vault Operations](#vault-operations)
- [Monitoring Stack](#monitoring-stack)
- [Backup & Restore](#backup--restore)
- [Logging Stack](#logging-stack)
- [Service Mesh (Istio)](#service-mesh-istio)
- [Troubleshooting](#troubleshooting)
- [File Location Quick Reference](#file-location-quick-reference)

---

## Cluster Access

### SSH to Nodes
```bash
# Gateway (jump host)
ssh root@192.168.0.21

# Nodes (via gateway)
ssh -J root@192.168.0.21 root@10.0.0.11  # node1 (external services)
ssh -J root@192.168.0.21 root@10.0.0.12  # node2 (k3s master)
ssh -J root@192.168.0.21 root@10.0.0.13  # node3 (k3s worker)
```

### Kubectl Access
```bash
# Create SSH tunnel to K3s API (run each time you need cluster access)
ssh -f -N -L 6443:10.0.0.12:6443 root@192.168.0.21

# Now kubectl works from your local machine
kubectl get nodes
kubectl get pods -A

# Check if tunnel is running
ps aux | grep "6443:10.0.0.12:6443"

# Kill tunnel when done
pkill -f "6443:10.0.0.12:6443"
```

### Access Web UIs via Port Forward
```bash
# Longhorn UI
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
# Open: http://localhost:8080

# Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Open: http://localhost:9090

# Grafana (when enabled)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Open: http://localhost:3000
```

### Access Vault/Minio on node1
```bash
# Vault UI
ssh -L 8200:10.0.0.11:8200 root@192.168.0.21
# Open: https://localhost:8200
# Root token: cat /etc/vault/unseal.json on node1

# Minio UI
ssh -L 9091:10.0.0.11:9091 root@192.168.0.21
# Open: http://localhost:9091
# Credentials: stored in Vault
```

---

## GitOps - Adding/Updating Apps

### Directory Structure
```
kubernetes/
├── clusters/prod/           # What FluxCD watches
│   ├── infra/              # Platform services
│   │   ├── kustomization.yaml       # Lists all infra apps
│   │   └── *-app.yaml              # FluxCD Kustomization per app
│   ├── apps/               # Your applications
│   │   ├── kustomization.yaml
│   │   └── *-app.yaml
│   └── config/
│       └── cluster-settings.yaml   # Global variables
├── platform/               # Infrastructure app manifests
│   └── [app-name]/
│       ├── app/           # Deployment manifests
│       │   ├── base/
│       │   └── overlays/prod/
│       └── config/        # Configuration (secrets, settings)
│           ├── base/
│           └── overlays/prod/
└── apps/                   # Application manifests
    └── [app-name]/
        └── app/
            ├── base/
            └── overlays/prod/
```

### Add a New Application

**Method 1: Simple manifest app**
```bash
# 1. Create directory structure
mkdir -p kubernetes/apps/my-app/app/{base,overlays/prod}

# 2. Create namespace and manifests in base/
cat > kubernetes/apps/my-app/app/base/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: my-app
resources:
- ns.yaml
- deployment.yaml
- service.yaml
EOF

cat > kubernetes/apps/my-app/app/base/ns.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
EOF

# 3. Create prod overlay
cat > kubernetes/apps/my-app/app/overlays/prod/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
# Add prod-specific patches/resources here
EOF

# 4. Create FluxCD Kustomization
cat > kubernetes/clusters/prod/apps/my-app-app.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-app
  namespace: flux-system
spec:
  interval: 30m
  targetNamespace: my-app
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./kubernetes/apps/my-app/app/overlays/prod
  prune: true
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-settings
EOF

# 5. Register in kustomization.yaml
echo "  - my-app-app.yaml" >> kubernetes/clusters/prod/apps/kustomization.yaml

# 6. Commit and push
git add kubernetes/
git commit -m "feat: add my-app application"
git push

# FluxCD will detect and deploy within ~1 minute
```

**Method 2: Helm chart app**
```bash
# 1. Create HelmRepository (if not exists)
cat > kubernetes/clusters/prod/repositories/helm/my-helm-repo.yaml <<EOF
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: my-helm-repo
  namespace: flux-system
spec:
  interval: 24h
  url: https://charts.example.com
EOF

# 2. Create HelmRelease
cat > kubernetes/platform/my-app/app/base/helm.yaml <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: my-app
spec:
  interval: 30m
  chart:
    spec:
      chart: my-app-chart
      version: "1.2.3"
      sourceRef:
        kind: HelmRepository
        name: my-helm-repo
        namespace: flux-system
  values:
    # Override helm values here
    replicaCount: 2
EOF

# 3. Follow steps 4-6 from Method 1
```

### Update an Existing App
```bash
# Edit the manifest
vim kubernetes/platform/longhorn/app/overlays/prod/values.yaml

# Commit and push
git add kubernetes/platform/longhorn/
git commit -m "fix: update longhorn configuration"
git push

# FluxCD will reconcile automatically, or force immediate sync:
kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

### Temporarily Disable an App
```bash
# Comment it out in kustomization.yaml
vim kubernetes/clusters/prod/infra/kustomization.yaml
# Change:  - grafana-app.yaml
# To:      # - grafana-app.yaml

git commit -am "chore: disable grafana temporarily"
git push
```

### Use Cluster Variables in Manifests
Variables from `kubernetes/clusters/prod/config/cluster-settings.yaml` are available:
```yaml
# Available variables:
# ${CLUSTER_DOMAIN}                  = homelab.jacobgade.com
# ${S3_BACKUP_SERVER}                = s3.jacobgade.com
# ${EXTERNAL_DNS_SERVER}             = 10.0.0.11
# ${NGINX_LOAD_BALANCER_IP}          = 10.0.0.65
# ${FLUENTD_LOAD_BALANCER_IP}        = 10.0.0.66
# ${ISTIO_GATEWAY_LOAD_BALANCER_IP}  = 10.0.0.67

# Example usage in manifest:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  rules:
  - host: my-app.${CLUSTER_DOMAIN}  # Becomes: my-app.homelab.jacobgade.com
```

---

## Certificate Management

### Architecture Overview
```
Vault (node1)
  └─→ ExternalSecret pulls → K8s Secret (cloudflare-api-token)
        └─→ ClusterIssuer (letsencrypt-issuer) uses secret
              └─→ Ingress annotation triggers → cert-manager
                    └─→ ACME DNS-01 challenge → Certificate issued
```

### Add TLS to an Ingress
```yaml
# In your app's values.yaml or ingress manifest:
ingress:
  enabled: true
  ingressClassName: nginx
  host: myapp.${CLUSTER_DOMAIN}
  tls: true
  tlsSecret: myapp-tls  # Name for the certificate secret

  annotations:
    # Trigger cert-manager to create certificate
    cert-manager.io/cluster-issuer: letsencrypt-issuer
    cert-manager.io/common-name: myapp.${CLUSTER_DOMAIN}
```

### Check Certificate Status
```bash
# List all certificates
kubectl get certificate -A

# Check specific certificate details
kubectl describe certificate longhorn-tls -n longhorn-system

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager -f

# Check certificate secret
kubectl get secret longhorn-tls -n longhorn-system -o yaml
```

### Troubleshoot Certificate Issues
```bash
# Check CertificateRequest
kubectl get certificaterequest -A
kubectl describe certificaterequest <name> -n <namespace>

# Check ACME Challenge
kubectl get challenge -A
kubectl describe challenge <name> -n <namespace>

# Force certificate renewal (delete and recreate)
kubectl delete certificate longhorn-tls -n longhorn-system
# cert-manager will recreate automatically

# Check ClusterIssuer status
kubectl describe clusterissuer letsencrypt-issuer

# Verify DNS record during challenge
dig _acme-challenge.longhorn.homelab.jacobgade.com TXT
```

### Update Cloudflare API Token
```bash
# 1. Update in Vault on node1
ssh -J root@192.168.0.21 root@10.0.0.11
vault kv put secret/certmanager/cloudflare api_token="new-token-here"

# 2. Force ExternalSecret to sync
kubectl annotate externalsecret cloudflare-api-token -n cert-manager \
  force-sync=$(date +%s) --overwrite

# 3. Restart cert-manager to pick up new secret
kubectl rollout restart deployment cert-manager -n cert-manager
```

### Available Issuers
```bash
# Production Let's Encrypt (rate limited)
cert-manager.io/cluster-issuer: letsencrypt-issuer

# Self-signed (for testing)
cert-manager.io/cluster-issuer: self-signed-issuer

# CA-signed (your own CA)
cert-manager.io/cluster-issuer: ca-issuer
```

### Key Files
```
kubernetes/platform/cert-manager/
├── config/overlays/prod/
│   ├── cloudflare-externalsecret.yaml    # Pulls from Vault
│   ├── ionos-issuer.yaml                 # ClusterIssuer definition
│   └── kustomization.yaml
└── app/overlays/prod/                    # cert-manager deployment
```

---

## Storage Operations

### Architecture
```
node1 (External SAN)
  └─→ NVMe SSD (200GB)
       ├─→ iSCSI LUN1 (100GB) → node2:/storage
       └─→ iSCSI LUN2 (100GB) → node3:/storage
              └─→ Longhorn uses /storage for PV backing

Pod → PVC → Longhorn StorageClass → Volume on /storage → iSCSI → node1 SSD
```

### Check Storage Status
```bash
# Longhorn volumes
kubectl get pv
kubectl get pvc -A

# Check Longhorn system
kubectl get pods -n longhorn-system

# Check node storage
ssh -J root@192.168.0.21 root@10.0.0.12 "df -h /storage"
ssh -J root@192.168.0.21 root@10.0.0.13 "df -h /storage"

# iSCSI connections
ssh -J root@192.168.0.21 root@10.0.0.12 "iscsiadm -m session"
```

### Create a PersistentVolumeClaim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce  # Or ReadWriteMany for shared storage
  storageClassName: longhorn  # Default storage class
  resources:
    requests:
      storage: 10Gi
```

### Troubleshoot Storage Issues
```bash
# Check Longhorn manager logs
kubectl logs -n longhorn-system deployment/longhorn-manager

# Check iSCSI connectivity from K8s nodes
ssh -J root@192.168.0.21 root@10.0.0.12 "systemctl status iscsid"
ssh -J root@192.168.0.21 root@10.0.0.12 "iscsiadm -m session -P 3"

# Check SAN on node1
ssh -J root@192.168.0.21 root@10.0.0.11
tgtadm --mode target --op show  # Show iSCSI targets
lvs  # Show LVM volumes
df -h  # Check disk space
```

### Backup/Restore with Longhorn
```bash
# Access Longhorn UI
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
# Open http://localhost:8080

# Configure backup target (Minio S3)
# Settings → Backup Target → s3://bucket-name@region/path
# Settings → Backup Target Credential Secret → minio-secret

# Create backup (via UI or CLI)
kubectl create -f - <<EOF
apiVersion: longhorn.io/v1beta2
kind: BackupVolume
metadata:
  name: pvc-backup
  namespace: longhorn-system
EOF
```

### Key Configuration
```
kubernetes/platform/longhorn/app/overlays/prod/values.yaml
  defaultSettings:
    defaultDataPath: "/storage"  # Use iSCSI mount
```

---

## Vault Operations

### Access Vault
```bash
# SSH tunnel
ssh -L 8200:10.0.0.11:8200 apollo@192.168.0.21
# Open https://localhost:8200

# Get root token
ssh -J root@192.168.0.21 apollo@10.0.0.11 "cat /etc/vault/unseal.json"

# CLI access (from node1)
ssh -J root@192.168.0.21 apollo@10.0.0.11
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='<root-token>'
vault status
```

### Common Secret Paths
```bash
# Cert-manager Cloudflare token
vault kv get secret/certmanager/cloudflare
vault kv put secret/certmanager/cloudflare api_token="your-token"

# Minio credentials
vault kv get secret/minio

# External-secrets references these as:
# remoteRef:
#   key: certmanager/cloudflare
#   property: api_token
```

### Add a New Secret
```bash
# 1. Store in Vault
vault kv put secret/myapp/database \
  username="dbuser" \
  password="secretpass"

# 2. Create ExternalSecret in K8s
cat > kubernetes/platform/myapp/config/base/externalsecret.yaml <<EOF
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: myapp-db-secret
  namespace: myapp
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: myapp-db-secret
  data:
  - secretKey: username
    remoteRef:
      key: myapp/database
      property: username
  - secretKey: password
    remoteRef:
      key: myapp/database
      property: password
EOF

# 3. Commit and push
git add kubernetes/platform/myapp/
git commit -m "feat: add database secret for myapp"
git push
```

### Troubleshoot External Secrets
```bash
# Check ExternalSecret status
kubectl get externalsecret -A
kubectl describe externalsecret cloudflare-api-token -n cert-manager

# Check ClusterSecretStore
kubectl get clustersecretstore
kubectl describe clustersecretstore vault-backend

# Verify secret was created
kubectl get secret myapp-db-secret -n myapp -o yaml

# Check external-secrets-operator logs
kubectl logs -n external-secrets deployment/external-secrets -f
```

---

## Monitoring Stack

Your cluster runs **kube-prometheus-stack** (Prometheus + Alertmanager + Grafana) for comprehensive monitoring.

### Architecture
```
Metrics Sources (nodes, pods, services)
    ↓
Prometheus (scrapes metrics every 30s)
    ↓ (stores)
TSDB (time-series database)
    ↓ (queries)
Grafana (dashboards) + Alertmanager (alerts)
```

### Access Monitoring Services

**Via Public Ingress (OAuth2 disabled):**
```bash
# Prometheus
https://monitoring.homelab.jacobgade.com/prometheus

# Alertmanager
https://monitoring.homelab.jacobgade.com/alertmanager

# Grafana (currently disabled - Tempo dependency)
https://grafana.homelab.jacobgade.com
```

**Via Port Forward:**
```bash
# Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Open: http://localhost:9090

# Alertmanager
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
# Open: http://localhost:9093

# Grafana (when enabled)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Open: http://localhost:3000
# Default credentials: admin / prom-operator
```

### Common Prometheus Queries

```promql
# CPU usage by pod
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod, namespace)

# Memory usage by pod
sum(container_memory_working_set_bytes) by (pod, namespace)

# Node CPU usage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Node memory usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Pod restarts in last hour
increase(kube_pod_container_status_restarts_total[1h]) > 0

# Persistent volume usage
(kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) * 100
```

### Check Monitoring Status

```bash
# Check all monitoring pods
kubectl get pods -n monitoring

# Check Prometheus targets (what's being monitored)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Open: http://localhost:9090/targets

# Check ServiceMonitors (defines what to scrape)
kubectl get servicemonitor -n monitoring

# Check PrometheusRules (alerting rules)
kubectl get prometheusrule -n monitoring
```

### Configure Alertmanager

**Current status:** Alertmanager is running but not configured to send notifications.

```bash
# Edit Alertmanager config
kubectl edit secret alertmanager-kube-prometheus-stack-alertmanager -n monitoring

# Or update via values.yaml:
# kubernetes/platform/kube-prometheus-stack/app/components/alertmanager/values.yaml
```

**Example: Slack notifications**
```yaml
alertmanager:
  config:
    global:
      slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
    route:
      receiver: 'slack'
      group_by: ['alertname', 'cluster', 'service']
    receivers:
    - name: 'slack'
      slack_configs:
      - channel: '#alerts'
        title: 'Cluster Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

### Add Custom Monitoring to Your App

```yaml
# Add ServiceMonitor to your app
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: my-app
  labels:
    release: kube-prometheus-stack  # Important!
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Enable Grafana

Currently disabled due to Tempo dependency. To enable:

```bash
# 1. Remove disable-grafana component from kustomization
vim kubernetes/clusters/prod/infra/kube-prometheus-stack-app.yaml

# 2. Either: enable Tempo OR remove Tempo datasource from Grafana config

# 3. Commit and push
git commit -am "feat: enable Grafana"
git push
```

### Key Files
```
kubernetes/platform/kube-prometheus-stack/
├── app/
│   ├── components/
│   │   ├── ingress/values.yaml          # Ingress configuration
│   │   ├── disable-grafana/values.yaml  # Grafana disable flag
│   │   └── k3s/values.yaml              # K3s-specific settings
│   └── overlays/prod/
└── externalnodes-monitoring/            # Monitor external node (node1)
```

---

## Backup & Restore

Your cluster has **two backup layers**: OS-level (Restic) and Kubernetes-level (Velero).

### Backup Architecture

```
OS Backups (Restic)
  ├─→ /etc, /home, /var → Minio S3 (node1)
  └─→ Scheduled: systemd timer

K8s Backups (Velero - currently disabled)
  ├─→ Resources → Minio S3
  ├─→ Volumes → Longhorn snapshots → Minio S3
  └─→ Scheduled: CronJob
```

### OS Backups (Restic)

**Configure backups:**
```bash
# Initial setup (done via Ansible)
make configure-os-backup

# This configures:
# - Restic repository on Minio S3
# - systemd timer for automatic backups
# - Backup retention policy
```

**Manual backup:**
```bash
# Trigger backup on all nodes
make os-backup

# Or manually on specific node
ssh -J root@192.168.0.21 root@10.0.0.12
systemctl start restic-backup
```

**Check backup status:**
```bash
# Check systemd timer status
ssh -J root@192.168.0.21 root@10.0.0.12 "systemctl status restic-backup.timer"

# Check last backup
ssh -J root@192.168.0.21 root@10.0.0.12 "systemctl status restic-backup.service"

# List backups (requires restic config)
ssh -J root@192.168.0.21 root@10.0.0.12
export RESTIC_REPOSITORY="s3:https://s3.jacobgade.com/restic-backups"
export RESTIC_PASSWORD="<from-vault>"
restic snapshots
```

**Restore from backup:**
```bash
# List available snapshots
restic snapshots

# Restore specific snapshot
restic restore <snapshot-id> --target /tmp/restore

# Restore specific path
restic restore latest --target /tmp/restore --path /etc/kubernetes
```

### Kubernetes Backups (Velero)

**Current status:** Velero is **disabled** to save resources. Enable when needed.

**Architecture:**
```
Velero Components:
├─→ Server: Backup controller
├─→ CSI Plugin: Volume snapshots
├─→ AWS Plugin: S3 storage
└─→ Storage: Minio S3 on node1
```

**Enable Velero:**
```bash
# 1. Uncomment in infra kustomization
vim kubernetes/clusters/prod/infra/kustomization.yaml
# Change: # - velero-app.yaml
# To:     - velero-app.yaml

# 2. Commit and push
git commit -am "feat: enable Velero"
git push
```

**Create a backup:**
```bash
# Backup entire cluster
velero backup create full-backup

# Backup specific namespace
velero backup create app-backup --include-namespaces my-app

# Backup with volume snapshots
velero backup create full-backup --snapshot-volumes=true

# Scheduled backup (via CRD)
cat <<EOF | kubectl apply -f -
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  template:
    includedNamespaces:
    - '*'
    snapshotVolumes: true
    ttl: 720h0m0s  # 30 days retention
EOF
```

**List and restore backups:**
```bash
# List backups
velero backup get

# Get backup details
velero backup describe full-backup

# Restore from backup
velero restore create --from-backup full-backup

# Restore specific namespace
velero restore create --from-backup full-backup \
  --include-namespaces my-app

# Check restore status
velero restore get
velero restore describe <restore-name>
```

**Backup Configuration:**
```bash
# Backup location
kubectl get backupstoragelocation -n velero

# Check Velero status
velero backup-location get

# Test backup location connectivity
velero backup-location get default -o json | jq .status
```

### Disaster Recovery Scenarios

**Scenario 1: Single pod data loss**
```bash
# Restore from Longhorn backup (via UI)
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
# UI → Volume → Create from Backup
```

**Scenario 2: Namespace accidentally deleted**
```bash
# Restore from Velero backup
velero restore create --from-backup full-backup \
  --include-namespaces deleted-namespace
```

**Scenario 3: Full cluster rebuild**
```bash
# 1. Reinstall K3s
make k3s-reset
make k3s-install
make k3s-bootstrap

# 2. Install Velero
# (uncomment velero-app.yaml and push)

# 3. Restore from backup
velero restore create full-restore --from-backup full-backup

# Note: GitOps will re-apply from Git automatically!
```

### Key Files
```
kubernetes/platform/velero/
├── app/
│   ├── base/values.yaml           # Velero configuration
│   └── overlays/prod/
└── config/
    └── overlays/prod/
        └── velero-externalsecret.yaml  # Minio credentials

ansible/backup_configuration.yml    # Restic setup playbook
```

---

## Logging Stack

Your cluster supports **two logging stacks**: EFK (Elasticsearch/Fluentd/Kibana) and LG (Loki/Grafana). Both are **currently disabled** to save resources.

### Architecture Options

**Option 1: EFK Stack (Heavy)**
```
Application Pods
    ↓ (logs to stdout)
Fluent-bit (DaemonSet on each node)
    ↓ (forwards)
Fluentd (aggregator)
    ↓ (stores)
Elasticsearch
    ↓ (queries)
Kibana (UI)
```

**Option 2: LG Stack (Lightweight - Recommended)**
```
Application Pods
    ↓ (logs to stdout)
Promtail (DaemonSet on each node)
    ↓ (forwards)
Loki (aggregator + storage)
    ↓ (queries)
Grafana (dashboards)
```

### Enable Logging Stack

**Option 1: Loki + Grafana (Recommended)**
```bash
# 1. Enable Loki
vim kubernetes/clusters/prod/infra/kustomization.yaml
# Uncomment: - loki-app.yaml

# 2. Enable Grafana (requires removing Tempo dependency)
vim kubernetes/clusters/prod/infra/kube-prometheus-stack-app.yaml
# Remove disable-grafana component

# 3. Commit and push
git commit -am "feat: enable Loki logging stack"
git push
```

**Option 2: EFK Stack (Heavy)**
```bash
# 1. Enable Elasticsearch
vim kubernetes/clusters/prod/infra/kustomization.yaml
# Uncomment: - elastic-stack-app.yaml

# 2. Enable Fluent-bit/Fluentd
# Uncomment: - fluent-app.yaml

# 3. Commit and push
git commit -am "feat: enable EFK logging stack"
git push

# Note: EFK requires ~2-3GB RAM minimum
```

### Access Logging UIs

**Loki (via Grafana):**
```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
# Open: http://localhost:3000
# Explore → Loki datasource
```

**Kibana (if EFK enabled):**
```bash
kubectl port-forward -n logging svc/kibana 5601:5601
# Open: http://localhost:5601
```

### Query Logs

**With Loki (LogQL):**
```logql
# All logs from namespace
{namespace="my-app"}

# Logs from specific pod
{namespace="my-app", pod="my-app-7d9f8c-abc"}

# Error logs only
{namespace="my-app"} |= "error"

# JSON parsing
{namespace="my-app"} | json | level="error"

# Rate of errors
rate({namespace="my-app"} |= "error" [5m])
```

**With Elasticsearch (Lucene):**
```
# Kibana Discover view
kubernetes.namespace:"my-app" AND log:"error"
```

### Add Logging to Your App

**For Loki:**
```yaml
# No special configuration needed!
# Just log to stdout/stderr
# Promtail automatically collects from all pods
```

**For custom log parsing (Loki):**
```yaml
# Add labels to your pod
metadata:
  labels:
    app: my-app
    version: v1.0
# These become searchable labels in Loki
```

### External Node Logging

node1 (external services node) logs are collected via Fluent-bit agent:

```bash
# Check Fluent-bit status on node1
ssh -J root@192.168.0.21 root@10.0.0.11 "systemctl status fluent-bit"

# View logs being collected
ssh -J root@192.168.0.21 root@10.0.0.11 "journalctl -u fluent-bit -f"
```

### Troubleshoot Logging

```bash
# Check log collectors
kubectl get pods -n logging  # or monitoring for Loki

# Check Loki status
kubectl logs -n monitoring loki-0 -f

# Check Fluentd/Fluent-bit
kubectl logs -n logging fluentd-0 -f
kubectl logs -n logging fluent-bit-xxxxx -f

# Test log ingestion
kubectl run test-logger --image=busybox -- sh -c "while true; do echo 'Test log entry'; sleep 5; done"
# Check if logs appear in Loki/Elasticsearch
```

### Key Files
```
kubernetes/platform/loki/                 # Loki stack
kubernetes/platform/fluent/               # Fluentd + Fluent-bit
kubernetes/platform/elastic-stack/        # EFK stack

ansible/host_vars/node1.yml:86            # node1 Fluent-bit config
```

---

## Service Mesh (Istio)

Your cluster has **Istio** deployed for advanced traffic management, observability, and security.

### What is Istio?

Istio provides:
- **Traffic Management**: Canary deployments, A/B testing, traffic splitting
- **Observability**: Distributed tracing, metrics, service graphs
- **Security**: mTLS between services, authorization policies

### Architecture

```
Your App Pod
    ↓ (injected)
Envoy Sidecar Proxy
    ↓ (managed by)
Istio Control Plane (istiod)
    ↓ (routes through)
Istio Gateway (10.0.0.67)
    ↓
External Traffic
```

### Check Istio Status

```bash
# Check Istio components
kubectl get pods -n istio-system

# Check Istio version
kubectl get deployment istiod -n istio-system -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check gateways
kubectl get gateway -A
kubectl get virtualservice -A

# Check which namespaces have sidecar injection
kubectl get namespace -L istio-injection
```

### Enable Istio for Your App

**Method 1: Namespace-level (Recommended)**
```bash
# Enable automatic sidecar injection for namespace
kubectl label namespace my-app istio-injection=enabled

# Restart pods to inject sidecars
kubectl rollout restart deployment -n my-app
```

**Method 2: Pod-level**
```yaml
# Add annotation to pod spec
metadata:
  annotations:
    sidecar.istio.io/inject: "true"
```

### Expose App via Istio Gateway

```yaml
---
# Gateway (entry point)
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-app-gateway
  namespace: my-app
spec:
  selector:
    istio: gateway  # Use default Istio gateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "myapp.homelab.jacobgade.com"
---
# VirtualService (routing)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
  namespace: my-app
spec:
  hosts:
  - "myapp.homelab.jacobgade.com"
  gateways:
  - my-app-gateway
  http:
  - route:
    - destination:
        host: my-app
        port:
          number: 80
```

### Traffic Management Examples

**Canary Deployment (90/10 split):**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-canary
spec:
  hosts:
  - my-app
  http:
  - route:
    - destination:
        host: my-app
        subset: v1
      weight: 90
    - destination:
        host: my-app
        subset: v2  # Canary version
      weight: 10
```

**Circuit Breaking:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-app
spec:
  host: my-app
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

### Observability with Kiali

**Current status:** Kiali is **disabled** (requires Keycloak authentication).

**Enable Kiali:**
```bash
# 1. Deploy Keycloak OR disable auth in Kiali config
vim kubernetes/clusters/prod/infra/kustomization.yaml
# Uncomment: - kiali-app.yaml

# 2. Access Kiali
kubectl port-forward -n istio-system svc/kiali 20001:20001
# Open: http://localhost:20001
```

**What Kiali shows:**
- Service mesh topology (visual graph)
- Request rates, latencies, error rates
- Traffic flows between services
- Configuration validation

### Access Istio Gateway Externally

```bash
# Gateway LoadBalancer IP (from cluster-settings)
ISTIO_GATEWAY_IP=10.0.0.67

# Add DNS record (or use /etc/hosts)
echo "$ISTIO_GATEWAY_IP myapp.homelab.jacobgade.com" | sudo tee -a /etc/hosts

# Test access
curl http://myapp.homelab.jacobgade.com
```

### Troubleshoot Istio

```bash
# Check if sidecar is injected
kubectl get pod my-app-xxx -o jsonpath='{.spec.containers[*].name}'
# Should show: my-app, istio-proxy

# Check sidecar logs
kubectl logs my-app-xxx -c istio-proxy

# Describe VirtualService
kubectl describe virtualservice my-app -n my-app

# Check Istio configuration sync
istioctl proxy-status

# Analyze configuration issues
istioctl analyze -A
```

### Key Files
```
kubernetes/platform/istio/
├── app/                        # Istio control plane
└── gateway/                    # Istio gateway

kubernetes/apps/book-info/      # Example Istio-enabled app
```

---

## Troubleshooting

### FluxCD Issues

**Check FluxCD status:**
```bash
# Overall status
kubectl get kustomization -n flux-system
kubectl get gitrepository -n flux-system

# Check specific app
kubectl describe kustomization longhorn-app -n flux-system

# Check FluxCD logs
kubectl logs -n flux-system deployment/source-controller -f
kubectl logs -n flux-system deployment/kustomize-controller -f
```

**Force reconciliation:**
```bash
# Force GitRepository sync
kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite

# Force Kustomization sync
kubectl annotate kustomization longhorn-app -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

**Suspend/Resume FluxCD resources:**
```bash
# Suspend (stop reconciliation)
flux suspend kustomization longhorn-app

# Resume
flux resume kustomization longhorn-app
```

### Pod Issues

```bash
# Get pod status
kubectl get pods -A

# Check pod logs
kubectl logs -n monitoring prometheus-0 -f

# Previous logs (after crash)
kubectl logs -n monitoring prometheus-0 --previous

# Describe pod (events, status)
kubectl describe pod prometheus-0 -n monitoring

# Execute into pod
kubectl exec -it prometheus-0 -n monitoring -- /bin/sh

# Check pod resource usage
kubectl top pod -A
kubectl top node
```

### Namespace Stuck Deleting

```bash
# Remove finalizers
kubectl get namespace <namespace> -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw "/api/v1/namespaces/<namespace>/finalize" -f -

# Or for specific resource
kubectl patch pvc my-pvc -n my-namespace -p '{"metadata":{"finalizers":null}}'
```

### Network/Ingress Issues

```bash
# Check ingress
kubectl get ingress -A
kubectl describe ingress longhorn -n longhorn-system

# Check NGINX ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller -f

# Check Cilium (CNI)
kubectl get pods -n kube-system -l k8s-app=cilium
kubectl exec -n kube-system cilium-xxxxx -- cilium status

# Check LoadBalancer services
kubectl get svc -A | grep LoadBalancer

# Test DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup longhorn-frontend.longhorn-system.svc.cluster.local
```

### Cluster Health

```bash
# Node status
kubectl get nodes -o wide
kubectl describe node node2

# System pods
kubectl get pods -n kube-system

# Resource usage
kubectl top nodes
kubectl top pods -A --sort-by=memory

# Check etcd (on master node)
ssh -J root@192.168.0.21 root@10.0.0.12
k3s kubectl get --raw /livez
k3s kubectl get --raw /readyz
```

### Ansible Operations

```bash
# Test connectivity
cd ~/Documents/Programming/Repos/homelab
make -C ansible-runner  # Setup runner first
ansible-runner/ansible-runner.sh ansible -i ansible/inventory.yml all -m ping

# Update all nodes
make os-upgrade

# Reinstall K3s (nuclear option)
make k3s-reset
make k3s-install
make k3s-bootstrap
```

---

## File Location Quick Reference

### Where to Find Things

| What | Where |
|------|-------|
| **Add new platform service** | `kubernetes/clusters/prod/infra/kustomization.yaml` |
| **Add new application** | `kubernetes/clusters/prod/apps/kustomization.yaml` |
| **Global variables** | `kubernetes/clusters/prod/config/cluster-settings.yaml` |
| **Certificate issuers** | `kubernetes/platform/cert-manager/config/overlays/prod/` |
| **Node inventory** | `ansible/inventory.yml` |
| **Node-specific config** | `ansible/host_vars/node1.yml`, etc. |
| **Cluster-wide Ansible vars** | `ansible/group_vars/all.yml` |
| **Encrypted secrets (Ansible)** | `ansible/vars/vault.yml` |
| **Cloud-init configs** | `metal/rpi/cloud-init/node1/user-data-centralizedSAN` |
| **Longhorn storage config** | `kubernetes/platform/longhorn/app/overlays/prod/values.yaml` |
| **NGINX ingress config** | `kubernetes/platform/nginx/app/overlays/prod/` |
| **Makefile commands** | `./Makefile` (root) |

### Common Service Ingresses

| Service | URL | Auth |
|---------|-----|------|
| Longhorn UI | https://longhorn.homelab.jacobgade.com | OAuth2 disabled |
| Prometheus | https://prometheus.homelab.jacobgade.com | OAuth2 disabled |
| Alertmanager | https://alertmanager.homelab.jacobgade.com | OAuth2 disabled |
| Hubble (Cilium) | https://hubble.homelab.jacobgade.com | OAuth2 disabled |
| Grafana | https://grafana.homelab.jacobgade.com | Disabled (Tempo dep) |
| Kiali | https://kiali.homelab.jacobgade.com | Disabled (Keycloak dep) |

### Service Endpoints (Internal)

| Service | Endpoint |
|---------|----------|
| **Vault** | http://10.0.0.11:8200 |
| **Minio UI** | http://10.0.0.11:9091 |
| **DNS** | 10.0.0.11:53 |
| **HAProxy** | 10.0.0.11 |
| **K3s API** | https://10.0.0.12:6443 |
| **NGINX LB** | 10.0.0.65 |
| **Fluentd LB** | 10.0.0.66 |
| **Istio Gateway LB** | 10.0.0.67 |

### Makefile Commands

```bash
# Initial setup (full cluster)
make external-setup        # Setup node1 (Vault, Minio, DNS)
make nodes-setup           # Setup K3s nodes
make k3s-install          # Install K3s
make k3s-bootstrap        # Bootstrap with FluxCD

# Maintenance
make os-upgrade           # Update all nodes
make configure-os-backup  # Setup Restic backups

# Reset (nuclear options)
make k3s-reset            # Reset K3s cluster
make external-services-reset  # Reset external services

# Utilities
make shutdown-picluster   # Shutdown all nodes
make get-pi-status        # Check Pi throttling
```

---

## Quick Debugging Workflows

### "My app isn't deploying"
```bash
# 1. Check if FluxCD sees it
kubectl get kustomization -n flux-system | grep myapp

# 2. Check status
kubectl describe kustomization myapp-app -n flux-system

# 3. Check the Git repo
kubectl describe gitrepository flux-system -n flux-system

# 4. Force sync
kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite

# 5. Check pod status
kubectl get pods -n myapp
kubectl describe pod myapp-xxx -n myapp
kubectl logs -n myapp myapp-xxx
```

### "Certificate not working"
```bash
# 1. Check certificate
kubectl get certificate -A | grep myapp
kubectl describe certificate myapp-tls -n myapp

# 2. Check challenge
kubectl get challenge -A

# 3. Check DNS (if DNS-01)
dig _acme-challenge.myapp.homelab.jacobgade.com TXT

# 4. Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager -f

# 5. Recreate certificate
kubectl delete certificate myapp-tls -n myapp
```

### "Storage not mounting"
```bash
# 1. Check PVC
kubectl get pvc -n myapp
kubectl describe pvc myapp-data -n myapp

# 2. Check Longhorn
kubectl get pods -n longhorn-system

# 3. Check iSCSI on nodes
ssh -J root@192.168.0.21 root@10.0.0.12 "iscsiadm -m session"

# 4. Check /storage mount
ssh -J root@192.168.0.21 root@10.0.0.12 "df -h /storage"
```

### "Can't access cluster"
```bash
# 1. Check SSH tunnel
ps aux | grep "6443:10.0.0.12:6443"

# 2. Recreate tunnel
pkill -f "6443:10.0.0.12:6443"
ssh -f -N -L 6443:10.0.0.12:6443 root@192.168.0.21

# 3. Test kubectl
kubectl get nodes

# 4. Check K3s on master
ssh -J root@192.168.0.21 root@10.0.0.12 "systemctl status k3s"
```

---

## Pro Tips

1. **Always commit before testing**: FluxCD only deploys from Git
2. **Use variables**: Leverage `${CLUSTER_DOMAIN}` instead of hardcoding
3. **Check FluxCD first**: Most issues are in GitRepository or Kustomization status
4. **Tunnel is temporary**: SSH tunnel dies on disconnect, recreate with `-f -N`
5. **Longhorn needs /storage**: Don't mess with the iSCSI mounts
6. **OAuth2 disabled**: Auth temporarily off, ingresses are public
7. **Prod overlays**: Your cluster uses `overlays/prod`, not `overlays/dev`
8. **Namespace matters**: ExternalSecrets must be in same namespace as the app using them

---

## Next Steps (from CLAUDE.md)

**High Priority:**
1. Fix gateway port forwarding (443 blocked)
2. Deploy authentication (Keycloak or ORY)
3. Enable OAuth2-proxy on ingresses

**Future:**
- Enable monitoring stack (Grafana)
- Deploy backup solution (Velero)
- Configure alerting (Alertmanager → email/Slack)
