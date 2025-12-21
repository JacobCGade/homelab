# Homelab Architecture Diagrams

Visual reference for understanding how all components in your K3s homelab cluster connect and interact.

> **Note:** These diagrams use [Mermaid](https://mermaid.js.org/) and will render automatically on GitHub and most modern markdown viewers.

## Table of Contents
- [Physical Architecture](#physical-architecture)
- [Network Topology](#network-topology)
- [Storage Architecture](#storage-architecture)
- [GitOps Flow](#gitops-flow)
- [Certificate Management Flow](#certificate-management-flow)
- [Request Flow (HTTPS)](#request-flow-https)
- [Monitoring Stack](#monitoring-stack)
- [Backup Architecture](#backup-architecture)
- [Component Dependencies](#component-dependencies)

---

## Physical Architecture

### Hardware and Node Layout

```mermaid
graph TB
    subgraph Internet["🌐 Internet / External Network (192.168.0.0/24)"]
        ISP[Internet Service Provider]
    end

    subgraph Gateway["🔥 OpenWrt Gateway (192.168.0.21)"]
        GW[OpenWrt Router<br/>- NAT & Port Forwarding<br/>- SSH Jump Host<br/>80/443 → 10.0.0.65]
    end

    subgraph Internal["🏠 Internal Network (10.0.0.0/24)"]
        subgraph Node1["📦 node1 (10.0.0.11) - External Services"]
            N1S[Services:<br/>- Bind9 DNS :53<br/>- Vault :8200<br/>- Minio S3 :9091<br/>- HAProxy<br/>- iSCSI Target]
            N1D[Storage:<br/>NVMe SSD 200GB<br/>├─ LUN1 100GB → node2<br/>└─ LUN2 100GB → node3]
        end

        subgraph Node2["⚙️ node2 (10.0.0.12) - K3s Master"]
            N2R[Raspberry Pi 5 8GB<br/>Roles:<br/>- K3s Control Plane<br/>- etcd<br/>- K8s API :6443]
            N2S[Storage:<br/>iSCSI from node1 LUN1<br/>Mounted: /storage 100GB]
            N2M[Resources:<br/>CPU: 4 cores ARM64<br/>RAM: 8GB 58% used]
        end

        subgraph Node3["⚙️ node3 (10.0.0.13) - K3s Worker"]
            N3R[Raspberry Pi 5 8GB<br/>Roles:<br/>- K3s Worker<br/>- Application Workloads]
            N3S[Storage:<br/>iSCSI from node1 LUN2<br/>Mounted: /storage 100GB]
            N3M[Resources:<br/>CPU: 4 cores ARM64<br/>RAM: 8GB 26% used]
        end

        subgraph LB["Load Balancer IPs (Cilium)"]
            LB1[10.0.0.65<br/>NGINX Ingress]
            LB2[10.0.0.66<br/>Fluentd]
            LB3[10.0.0.67<br/>Istio Gateway]
        end
    end

    ISP --> GW
    GW --> Node1
    GW --> Node2
    GW --> Node3

    N1D -.iSCSI.-> N2S
    N1D -.iSCSI.-> N3S

    Node2 --> LB
    Node3 --> LB

    style Node1 fill:#e1f5fe
    style Node2 fill:#fff3e0
    style Node3 fill:#fff3e0
    style Gateway fill:#ffebee
    style LB fill:#f3e5f5
```

**Cluster Summary:**
- **Total CPU:** 8 cores (ARM64)
- **Total RAM:** 16GB (6.7GB used, 9.3GB available)
- **Total Storage:** 200GB (centralized on node1)

---

## Network Topology

### Complete Network Flow

```mermaid
graph TB
    subgraph External["External Network (192.168.0.0/24)"]
        DNS[Cloudflare DNS<br/>*.homelab.jacobgade.com<br/>→ 192.168.0.21]
        Gateway[OpenWrt Gateway<br/>192.168.0.21<br/>Port Forwarding:<br/>80→10.0.0.65:80<br/>443→10.0.0.65:443]
    end

    subgraph Internal["Internal Network (10.0.0.0/24)"]
        Node1[node1: 10.0.0.11<br/>DNS, Vault, Minio]
        Node2[node2: 10.0.0.12<br/>K3s Master]
        Node3[node3: 10.0.0.13<br/>K3s Worker]

        NGINX[NGINX Ingress<br/>LoadBalancer: 10.0.0.65]
        Istio[Istio Gateway<br/>LoadBalancer: 10.0.0.67]
    end

    subgraph K8s["Kubernetes Networking (Cilium CNI)"]
        PodCIDR[Pod Network<br/>10.42.0.0/16]
        SvcCIDR[Service Network<br/>10.43.0.0/16]
        CoreDNS[CoreDNS<br/>10.43.0.10<br/>*.svc.cluster.local]
    end

    DNS --> Gateway
    Gateway --> NGINX
    Gateway --> Istio
    Gateway --> Node1

    NGINX --> PodCIDR
    Istio --> PodCIDR
    PodCIDR --> SvcCIDR
    SvcCIDR --> CoreDNS

    Node2 --> K8s
    Node3 --> K8s

    style External fill:#ffebee
    style Internal fill:#e8f5e9
    style K8s fill:#e3f2fd
```

### DNS Resolution Flow

```mermaid
sequenceDiagram
    participant User as User's Browser
    participant CF as Cloudflare DNS
    participant GW as OpenWrt Gateway
    participant NGINX as NGINX Ingress
    participant CNI as Cilium CNI
    participant Pod as Application Pod

    User->>CF: Query longhorn.homelab.jacobgade.com
    CF-->>User: Returns 192.168.0.21
    User->>GW: HTTPS Request to :443
    GW->>NGINX: Forward to 10.0.0.65:443
    NGINX->>NGINX: Terminate TLS<br/>Check Host header
    NGINX->>CNI: Route to backend service
    CNI->>Pod: Deliver request
    Pod-->>CNI: Response
    CNI-->>NGINX: Response
    NGINX-->>GW: Encrypted response
    GW-->>User: Response

    Note over User,Pod: Total latency: ~30-50ms
```

---

## Storage Architecture

### Complete Storage Stack

```mermaid
graph TB
    subgraph Apps["Application Layer"]
        Pod1[Pod A<br/>volumeMount: /data]
        Pod2[Pod B<br/>volumeMount: /data]
        Pod3[Pod C<br/>volumeMount: /data]
    end

    subgraph K8s["Kubernetes Layer"]
        PVC1[PVC: my-data-1<br/>10Gi ReadWriteOnce]
        PVC2[PVC: my-data-2<br/>5Gi ReadWriteOnce]
        PVC3[PVC: my-data-3<br/>20Gi ReadWriteMany]

        SC[StorageClass: longhorn<br/>Provisioner: driver.longhorn.io<br/>ReclaimPolicy: Delete<br/>VolumeExpansion: true]

        PV1[PersistentVolume<br/>Managed by Longhorn<br/>Replicas: 2-3]
        PV2[PersistentVolume<br/>Managed by Longhorn<br/>Replicas: 2-3]
        PV3[PersistentVolume<br/>Managed by Longhorn<br/>Replicas: 2-3]
    end

    subgraph Longhorn["Longhorn Storage Layer"]
        LM[Longhorn Manager<br/>Volume orchestration<br/>Replication<br/>Snapshots<br/>Backup to S3]

        Node2Data[node2:/storage/longhorn/<br/>volume-1-replica-1<br/>volume-2-replica-1]
        Node3Data[node3:/storage/longhorn/<br/>volume-1-replica-2<br/>volume-3-replica-1]
    end

    subgraph iSCSI["iSCSI Layer"]
        Node2Mount[node2 (10.0.0.12)<br/>iSCSI Initiator<br/>/dev/sdb → /storage 100GB]
        Node3Mount[node3 (10.0.0.13)<br/>iSCSI Initiator<br/>/dev/sdb → /storage 100GB]

        iSCSITarget[node1 iSCSI Target<br/>iqn.2025-01.com.jacobgade<br/>├─ LUN1 100GB → node2<br/>└─ LUN2 100GB → node3]
    end

    subgraph Physical["Physical Storage"]
        SSD[node1 NVMe SSD<br/>200GB Total<br/>├─ Root: 30GB<br/>└─ LVM: 170GB<br/>   ├─ LUN1: 100GB<br/>   └─ LUN2: 100GB]
    end

    Pod1 --> PVC1
    Pod2 --> PVC2
    Pod3 --> PVC3

    PVC1 --> SC
    PVC2 --> SC
    PVC3 --> SC

    SC --> PV1
    SC --> PV2
    SC --> PV3

    PV1 --> LM
    PV2 --> LM
    PV3 --> LM

    LM --> Node2Data
    LM --> Node3Data

    Node2Data --> Node2Mount
    Node3Data --> Node3Mount

    Node2Mount --> iSCSITarget
    Node3Mount --> iSCSITarget

    iSCSITarget --> SSD

    style Apps fill:#e1f5fe
    style K8s fill:#fff3e0
    style Longhorn fill:#f3e5f5
    style iSCSI fill:#e8f5e9
    style Physical fill:#ffebee
```

### Storage Data Flow

```mermaid
sequenceDiagram
    participant Pod as Application Pod
    participant K8s as Kubernetes
    participant LH as Longhorn Manager
    participant N2 as node2:/storage
    participant N3 as node3:/storage
    participant iSCSI as iSCSI (network)
    participant SSD as node1 NVMe SSD

    Pod->>K8s: Write 1MB file to PVC
    K8s->>LH: Write to Longhorn volume
    LH->>N2: Write primary replica
    LH->>N3: Write secondary replica
    N2->>iSCSI: Write to /storage
    N3->>iSCSI: Write to /storage
    iSCSI->>SSD: Write to LUN1
    iSCSI->>SSD: Write to LUN2
    SSD-->>iSCSI: ACK
    iSCSI-->>N2: ACK
    iSCSI-->>N3: ACK
    N2-->>LH: ACK
    N3-->>LH: ACK
    LH-->>K8s: Write complete
    K8s-->>Pod: File written

    Note over Pod,SSD: Data is replicated and persistent!
```

---

## GitOps Flow

### FluxCD Deployment Pipeline

```mermaid
graph TB
    subgraph Dev["👨‍💻 Developer Workflow"]
        Edit[Edit files in<br/>kubernetes/platform/myapp/]
        Commit[git commit -m<br/>'feat: update myapp']
        Push[git push origin master]
    end

    subgraph Git["📦 GitHub Repository"]
        Repo[github.com/JacobCGade/homelab<br/>Branch: master]
    end

    subgraph Flux["🔄 FluxCD (flux-system namespace)"]
        Source[Source Controller<br/>Polls every 1 minute]
        Kustomize[Kustomize Controller<br/>Processes manifests]
        Git2[GitRepository: flux-system<br/>Watches master branch]
    end

    subgraph Discover["🔍 Discovery"]
        ProdInfra[clusters/prod/infra/<br/>kustomization.yaml]
        AppKust[myapp-app.yaml<br/>FluxCD Kustomization]
        Manifests[platform/myapp/app/<br/>overlays/prod/]
    end

    subgraph Process["⚙️ Processing"]
        Vars[Inject variables from<br/>cluster-settings ConfigMap<br/>CLUSTER_DOMAIN, etc.]
        KustBuild[Build with Kustomize<br/>base + prod overlays]
        Final[Final manifests ready]
    end

    subgraph K8s["☸️ Kubernetes"]
        API[Apply to K8s API]
        Scheduler[Schedule pods]
        Running[Pod running!]
    end

    Edit --> Commit
    Commit --> Push
    Push --> Repo

    Repo --> Source
    Source --> Git2
    Git2 --> Kustomize

    Kustomize --> ProdInfra
    ProdInfra --> AppKust
    AppKust --> Manifests

    Manifests --> Vars
    Vars --> KustBuild
    KustBuild --> Final

    Final --> API
    API --> Scheduler
    Scheduler --> Running

    style Dev fill:#e3f2fd
    style Git fill:#fff3e0
    style Flux fill:#f3e5f5
    style Discover fill:#e8f5e9
    style Process fill:#fff9c4
    style K8s fill:#ffebee
```

### Directory Structure and Discovery

```mermaid
graph LR
    subgraph Repo["Git Repository Structure"]
        Root[kubernetes/]

        Clusters[clusters/prod/]
        Infra[infra/kustomization.yaml]
        Apps[apps/kustomization.yaml]
        Config[config/cluster-settings.yaml]

        Platform[platform/]
        MyApp[myapp/]
        AppDir[app/]
        Base[base/]
        Overlays[overlays/prod/]

        Root --> Clusters
        Clusters --> Infra
        Clusters --> Apps
        Clusters --> Config

        Root --> Platform
        Platform --> MyApp
        MyApp --> AppDir
        AppDir --> Base
        AppDir --> Overlays
    end

    subgraph Flux["FluxCD Watches"]
        Watch[FluxCD watches<br/>clusters/prod/]
        Reads[Reads infra/<br/>kustomization.yaml]
        Discovers[Discovers<br/>myapp-app.yaml]
        Points[Points to<br/>platform/myapp/.../prod/]
        Deploys[Deploys to cluster]
    end

    Clusters -.watches.-> Watch
    Infra -.reads.-> Reads
    Reads --> Discovers
    Overlays -.reads.-> Points
    Points --> Deploys

    style Repo fill:#e3f2fd
    style Flux fill:#f3e5f5
```

---

## Certificate Management Flow

### Complete TLS Certificate Lifecycle

```mermaid
sequenceDiagram
    participant Vault as Vault (node1)
    participant ESO as External Secrets<br/>Operator
    participant Secret as K8s Secret<br/>cloudflare-api-token
    participant Issuer as ClusterIssuer<br/>letsencrypt-issuer
    participant Ingress as Ingress Resource<br/>with annotations
    participant CM as cert-manager
    participant LE as Let's Encrypt<br/>ACME Server
    participant CF as Cloudflare DNS
    participant NGINX as NGINX Ingress<br/>Controller

    Note over Vault,ESO: Step 1: Secret Management
    Vault->>ESO: ExternalSecret polls Vault
    ESO->>Vault: Read certmanager/cloudflare
    Vault-->>ESO: Returns api_token
    ESO->>Secret: Creates K8s Secret

    Note over Issuer,Secret: Step 2: ClusterIssuer Setup
    Issuer->>Secret: References cloudflare-api-token

    Note over Ingress,CM: Step 3: Certificate Request
    Ingress->>CM: Annotation triggers cert-manager
    Note over Ingress: cert-manager.io/cluster-issuer:<br/>letsencrypt-issuer

    CM->>CM: Create Certificate resource
    CM->>CM: Create CertificateRequest
    CM->>CM: Create Challenge (DNS-01)

    Note over CM,LE: Step 4: ACME Challenge
    CM->>CF: Create TXT record<br/>_acme-challenge.longhorn...
    CF-->>CM: DNS record created
    CM->>CM: Wait for DNS propagation (30-60s)
    CM->>LE: Notify: Ready for validation
    LE->>CF: Query TXT record
    CF-->>LE: Returns challenge token
    LE-->>CM: Domain verified ✓

    Note over LE,NGINX: Step 5: Certificate Issuance
    LE->>CM: Issue certificate + chain
    CM->>Secret: Store in longhorn-tls Secret
    NGINX->>Secret: Watch and load certificate

    Note over NGINX: Step 6: HTTPS Enabled!
    Note over NGINX: Site accessible with<br/>valid TLS certificate

    Note over CM: Auto-renewal in 60 days
```

### Certificate Flow Diagram

```mermaid
graph TB
    subgraph Vault["🔐 Vault Storage (node1)"]
        V[Vault KV Store<br/>certmanager/cloudflare<br/>api_token: xxx]
    end

    subgraph ESO["🔄 External Secrets"]
        E[ExternalSecret<br/>cloudflare-api-token]
        CS[ClusterSecretStore<br/>vault-backend]
        KS[K8s Secret<br/>cloudflare-api-token]
    end

    subgraph CertMgr["📜 cert-manager"]
        CI[ClusterIssuer<br/>letsencrypt-issuer<br/>Uses: cloudflare-api-token]
        Cert[Certificate<br/>longhorn-tls]
        Req[CertificateRequest]
        Chal[Challenge DNS-01]
    end

    subgraph ACME["🌐 Let's Encrypt"]
        LE[ACME Server<br/>Validates domain]
        CF[Cloudflare DNS<br/>TXT records]
    end

    subgraph App["📱 Application"]
        Ing[Ingress<br/>annotations trigger<br/>certificate creation]
        Sec[Secret: longhorn-tls<br/>tls.crt + tls.key]
        NGINX[NGINX uses cert<br/>for TLS termination]
    end

    V --> E
    E --> CS
    CS --> KS
    KS --> CI

    Ing --> Cert
    CI --> Cert
    Cert --> Req
    Req --> Chal

    Chal --> CF
    CF --> LE
    LE --> Cert

    Cert --> Sec
    Sec --> NGINX

    style Vault fill:#ffebee
    style ESO fill:#e8f5e9
    style CertMgr fill:#e3f2fd
    style ACME fill:#fff3e0
    style App fill:#f3e5f5
```

---

## Request Flow (HTTPS)

### External Request to Internal Service

```mermaid
sequenceDiagram
    participant User as 🌐 User's Browser
    participant DNS as Cloudflare DNS
    participant GW as OpenWrt Gateway<br/>(192.168.0.21)
    participant Cilium as Cilium LB<br/>(10.0.0.65)
    participant NGINX as NGINX Ingress<br/>Controller
    participant CoreDNS as CoreDNS
    participant Service as K8s Service<br/>longhorn-frontend
    participant Pod as Longhorn UI Pod

    User->>DNS: Query longhorn.homelab.jacobgade.com
    DNS-->>User: Returns 192.168.0.21

    User->>GW: HTTPS Request :443
    Note over User,GW: TLS encrypted

    GW->>Cilium: NAT forward to 10.0.0.65:443
    Note over GW,Cilium: Port forwarding rule

    Cilium->>NGINX: Route to NGINX pod
    Note over Cilium,NGINX: Cilium CNI routing

    NGINX->>NGINX: 1. Terminate TLS<br/>using longhorn-tls Secret
    NGINX->>NGINX: 2. Check Host header<br/>longhorn.homelab.jacobgade.com
    NGINX->>NGINX: 3. Match Ingress rule

    NGINX->>CoreDNS: Resolve longhorn-frontend<br/>.longhorn-system.svc
    CoreDNS-->>NGINX: Returns ClusterIP 10.43.x.x

    NGINX->>Service: HTTP to backend service
    Service->>Pod: Forward to pod

    Pod->>Pod: Process request<br/>Generate HTML

    Pod-->>Service: Response
    Service-->>NGINX: Response
    NGINX->>NGINX: Encrypt with TLS
    NGINX-->>Cilium: Encrypted response
    Cilium-->>GW: Response
    GW-->>User: Response

    Note over User,Pod: Total latency: 30-50ms
```

### Internal Pod-to-Pod Communication

```mermaid
graph LR
    subgraph PodA["Pod A (my-app namespace)"]
        A[Container makes<br/>HTTP request to<br/>prometheus.monitoring]
    end

    subgraph DNS["CoreDNS Resolution"]
        D[Query: prometheus.monitoring<br/>.svc.cluster.local<br/>Returns: ClusterIP 10.43.x.x]
    end

    subgraph Cilium["Cilium CNI"]
        C[Load balance across<br/>backend pods<br/>Route to pod IP 10.42.y.y]
    end

    subgraph PodB["Prometheus Pod"]
        B[Receive request<br/>Process PromQL query<br/>Return response]
    end

    A --> D
    D --> C
    C --> B
    B -.Response.-> C
    C -.Response.-> D
    D -.Response.-> A

    style PodA fill:#e3f2fd
    style DNS fill:#fff3e0
    style Cilium fill:#e8f5e9
    style PodB fill:#f3e5f5
```

---

## Monitoring Stack

### Observability Architecture

```mermaid
graph TB
    subgraph Sources["📊 Data Sources"]
        Apps[Application Pods<br/>/metrics endpoints]
        Kubelets[Kubelet<br/>Node metrics]
        KSM[kube-state-metrics<br/>K8s object metrics]
        NodeExp[node-exporter<br/>System metrics]
        ExtNode[node1<br/>External metrics]
    end

    subgraph Collection["📈 Metrics Collection"]
        SM[ServiceMonitors<br/>Define scrape targets]
        PM[PodMonitors<br/>Define pod targets]
        Prom[Prometheus<br/>Scrape every 30s<br/>Store in TSDB<br/>Retention: 15 days]
    end

    subgraph Visualization["📊 Visualization"]
        Grafana[Grafana<br/>Dashboards<br/>Currently disabled]
        PromUI[Prometheus UI<br/>PromQL queries]
    end

    subgraph Alerting["🚨 Alerting"]
        Rules[PrometheusRules<br/>Alert definitions]
        AM[Alertmanager<br/>Route & notify]
        Notify[Notifications:<br/>- Slack<br/>- Email<br/>- PagerDuty<br/>- Webhook]
    end

    Apps --> SM
    Kubelets --> SM
    KSM --> SM
    NodeExp --> PM
    ExtNode --> Prom

    SM --> Prom
    PM --> Prom

    Prom --> Grafana
    Prom --> PromUI

    Prom --> Rules
    Rules --> AM
    AM --> Notify

    style Sources fill:#e3f2fd
    style Collection fill:#fff3e0
    style Visualization fill:#e8f5e9
    style Alerting fill:#ffebee
```

### Metric Collection Flow

```mermaid
sequenceDiagram
    participant App as Application Pod<br/>/metrics
    participant SM as ServiceMonitor
    participant Prom as Prometheus
    participant TSDB as Time Series DB
    participant Grafana as Grafana
    participant Alert as Alertmanager

    Note over App: App exposes metrics
    App->>App: http_requests_total{method="GET"} 1543

    SM->>Prom: Configure scrape target<br/>app:8080/metrics every 30s

    loop Every 30 seconds
        Prom->>App: GET /metrics
        App-->>Prom: Prometheus format metrics
        Prom->>TSDB: Store metrics with timestamp
    end

    Note over Grafana: User views dashboard
    Grafana->>Prom: PromQL: rate(http_requests_total[5m])
    Prom->>TSDB: Query time series
    TSDB-->>Prom: Return data points
    Prom-->>Grafana: Return aggregated data

    Note over Alert: Alert evaluation
    Prom->>Prom: Evaluate rules every 1m
    Prom->>Prom: rate(http_requests_total[5m]) > 1000
    Prom->>Alert: Fire alert
    Alert->>Alert: Group & route alert
    Alert-->>Alert: Send to Slack
```

---

## Backup Architecture

### Multi-Layer Backup Strategy

```mermaid
graph TB
    subgraph Nodes["💻 Cluster Nodes"]
        N1[node1<br/>/etc, /home, /var]
        N2[node2<br/>/etc, /var/lib/rancher]
        N3[node3<br/>/etc, /var/lib/rancher]
    end

    subgraph OS["💾 OS Backups (Restic)"]
        Timer[Systemd Timer<br/>Daily at 2 AM]
        Restic[Restic Backup<br/>Encrypted & compressed<br/>Retention:<br/>- 7 daily<br/>- 4 weekly<br/>- 12 monthly]
    end

    subgraph K8s["☸️ Kubernetes Backups"]
        Velero[Velero<br/>Currently disabled<br/>Backs up:<br/>- All K8s resources<br/>- Volume snapshots]
    end

    subgraph Storage["💿 Volume Backups"]
        Longhorn[Longhorn<br/>Volume snapshots<br/>Incremental backups<br/>On-demand + scheduled]
    end

    subgraph Target["🎯 Backup Target"]
        Minio[Minio S3 on node1<br/>Buckets:<br/>- restic-backups/<br/>- k3s-velero/<br/>- longhorn-backups/]
        SSD[Stored on:<br/>node1 NVMe SSD<br/>~50GB backup data]
    end

    N1 --> Timer
    N2 --> Timer
    N3 --> Timer
    Timer --> Restic

    K8s --> Velero

    Storage --> Longhorn

    Restic --> Minio
    Velero --> Minio
    Longhorn --> Minio

    Minio --> SSD

    style Nodes fill:#e3f2fd
    style OS fill:#fff3e0
    style K8s fill:#e8f5e9
    style Storage fill:#f3e5f5
    style Target fill:#ffebee
```

### Disaster Recovery Workflow

```mermaid
graph TB
    Start[💥 Disaster: Complete cluster loss]

    Start --> Step1[1. Reinstall OS on nodes<br/>Using cloud-init configs]
    Step1 --> Step2[2. Run Ansible playbooks<br/>make external-setup<br/>make nodes-setup<br/>make k3s-install<br/>make k3s-bootstrap]
    Step2 --> Step3[3. GitOps auto-deploys<br/>FluxCD pulls from Git<br/>All apps deployed automatically]
    Step3 --> Step4[4. Enable Velero<br/>Uncomment in infra/kustomization.yaml]
    Step4 --> Step5[5. Restore from backup<br/>velero restore create<br/>--from-backup full-backup]
    Step5 --> Step6[6. Longhorn restores volumes<br/>Pods attach to restored PVCs]
    Step6 --> Done[✅ Cluster fully operational<br/>with all data intact!]

    Note1[Recovery time:<br/>30-60 minutes]

    Done -.-> Note1

    style Start fill:#ffebee
    style Done fill:#e8f5e9
    style Note1 fill:#fff3e0
```

---

## Component Dependencies

### Startup Order and Critical Paths

```mermaid
graph TB
    subgraph Level0["Level 0: Physical Infrastructure"]
        Boot[Node1, Node2, Node3 boot<br/>iSCSI connections established<br/>/storage mounted]
    end

    subgraph Level1["Level 1: External Services (node1)"]
        DNS[Bind9 DNS<br/>Required for name resolution]
        Vault[HashiCorp Vault<br/>Required for secrets]
        Minio[Minio S3<br/>Optional: for backups]
    end

    subgraph Level2["Level 2: K3s Core"]
        K3s[K3s Services<br/>- etcd<br/>- kube-apiserver<br/>- kube-scheduler<br/>- kubelet]
    end

    subgraph Level3["Level 3: CNI & DNS"]
        Cilium[Cilium CNI<br/>Pod networking & LB]
        CoreDNS[CoreDNS<br/>Service discovery]
    end

    subgraph Level4["Level 4: Core Platform"]
        Flux[FluxCD<br/>GitOps automation]
        ExtSec[External Secrets<br/>Vault → K8s Secrets]
        CertMgr[cert-manager<br/>TLS certificates]
        Ingress[NGINX Ingress<br/>HTTP routing]
    end

    subgraph Level5["Level 5: Storage"]
        CSI[CSI Snapshotter]
        Longhorn[Longhorn<br/>Distributed storage]
    end

    subgraph Level6["Level 6: Observability"]
        Prom[Prometheus<br/>Metrics collection]
        Metrics[metrics-server<br/>kubectl top]
    end

    subgraph Level7["Level 7: Optional Services"]
        Istio[Istio Service Mesh]
        Loki[Loki Logging]
        Velero[Velero Backups]
    end

    subgraph Level8["Level 8: Applications"]
        Apps[Your Applications]
    end

    Boot --> DNS
    Boot --> Vault
    Boot --> Minio
    DNS --> K3s
    Vault --> K3s

    K3s --> Cilium
    K3s --> CoreDNS

    Cilium --> Flux
    CoreDNS --> Flux

    Flux --> ExtSec
    ExtSec --> CertMgr
    Vault --> ExtSec
    CertMgr --> Ingress

    Cilium --> Longhorn
    Flux --> CSI
    CSI --> Longhorn

    Longhorn --> Prom
    Flux --> Prom
    Flux --> Metrics

    Prom --> Istio
    Prom --> Loki
    Prom --> Velero

    Longhorn --> Apps
    Ingress --> Apps
    Prom --> Apps

    style Level0 fill:#ffebee
    style Level1 fill:#fff3e0
    style Level2 fill:#e8f5e9
    style Level3 fill:#e3f2fd
    style Level4 fill:#f3e5f5
    style Level5 fill:#fce4ec
    style Level6 fill:#e0f2f1
    style Level7 fill:#f1f8e9
    style Level8 fill:#fff9c4
```

### Critical Failure Paths

```mermaid
graph TB
    subgraph Failures["💥 What Breaks What?"]
        F1[DNS node1 fails]
        F2[Vault fails]
        F3[Cilium fails]
        F4[Longhorn fails]
        F5[FluxCD fails]
    end

    subgraph Impact1["Impact: DNS Failure"]
        I1[CoreDNS can't resolve<br/>external names]
        I2[Pods can't pull images]
        I3[Cluster degraded]
    end

    subgraph Impact2["Impact: Vault Failure"]
        I4[external-secrets<br/>can't sync]
        I5[New Secrets not created]
        I6[cert-manager can't get<br/>Cloudflare token]
        I7[New certificates fail]
    end

    subgraph Impact3["Impact: Cilium Failure"]
        I8[Pod networking breaks]
        I9[Services unreachable]
        I10[Cluster non-functional]
    end

    subgraph Impact4["Impact: Longhorn Failure"]
        I11[New PVCs can't be<br/>provisioned]
        I12[Stateful apps can't start]
        I13[Existing volumes still work<br/>degraded mode]
    end

    subgraph Impact5["Impact: FluxCD Failure"]
        I14[GitOps stops]
        I15[No automatic deployments]
        I16[Existing apps still run<br/>manual management required]
    end

    F1 --> I1
    I1 --> I2
    I2 --> I3

    F2 --> I4
    I4 --> I5
    I5 --> I6
    I6 --> I7

    F3 --> I8
    I8 --> I9
    I9 --> I10

    F4 --> I11
    I11 --> I12
    I12 --> I13

    F5 --> I14
    I14 --> I15
    I15 --> I16

    style Failures fill:#ffebee
    style Impact1 fill:#fff3e0
    style Impact2 fill:#e3f2fd
    style Impact3 fill:#ffcdd2
    style Impact4 fill:#fff9c4
    style Impact5 fill:#e8f5e9
```

---

## Quick Reference

### Key Information

**IP Addresses:**
- Gateway/Jump Host: `192.168.0.21`
- node1 (External): `10.0.0.11` (Vault, Minio, DNS, iSCSI)
- node2 (K3s Master): `10.0.0.12`
- node3 (K3s Worker): `10.0.0.13`
- NGINX LoadBalancer: `10.0.0.65`
- Istio Gateway: `10.0.0.67`

**Access Methods:**
```bash
# SSH tunnel for K8s API
ssh -f -N -L 6443:10.0.0.12:6443 root@192.168.0.21

# SSH to nodes (via jump host)
ssh -J root@192.168.0.21 root@10.0.0.11  # node1
ssh -J root@192.168.0.21 root@10.0.0.12  # node2
ssh -J root@192.168.0.21 root@10.0.0.13  # node3

# Access Vault
ssh -L 8200:10.0.0.11:8200 root@192.168.0.21
# https://localhost:8200

# Access Minio
ssh -L 9091:10.0.0.11:9091 root@192.168.0.21
# http://localhost:9091
```

**Public Services:**
- Prometheus: `https://monitoring.homelab.jacobgade.com/prometheus`
- Alertmanager: `https://monitoring.homelab.jacobgade.com/alertmanager`
- Longhorn: `https://longhorn.homelab.jacobgade.com`
- Hubble: `https://hubble.homelab.jacobgade.com`

**Resources:**
- Total CPU: 8 cores (ARM64)
- Total RAM: 16GB (6.7GB used, 9.3GB available)
- Total Storage: 200GB (centralized SAN on node1)

---

## Additional Resources

- **CHEATSHEET.md**: Commands and operations reference
- **CLAUDE.md**: Project overview and current status
- **README.md**: Original picluster documentation

For detailed command references and troubleshooting workflows, see [CHEATSHEET.md](./CHEATSHEET.md).
