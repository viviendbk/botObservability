# Dofus Observability — Helm Umbrella Chart

Enterprise-grade observability pipeline for monitoring a fleet of automated Dofus game bots on a bare-metal `kubeadm` Kubernetes cluster.

## Architecture

```
┌─────────────┐     ┌───────────────┐     ┌──────────────────────┐
│  Dofus Bots │────▶│  Kafka        │────▶│  OTel Collector      │
│  (Producers)│     │  (bot_events) │     │  - Extract kamas     │
└─────────────┘     └───────────────┘     │    metric (gauge)    │
                                          │  - Forward raw logs  │
                                          └──────┬───────┬───────┘
                                                 │       │
                                     ┌───────────┘       └───────────┐
                                     ▼                               ▼
                              ┌──────────────┐              ┌──────────────┐
                              │  Prometheus  │              │     Loki     │
                              │  (metrics)   │              │   (logs)     │
                              └──────┬───────┘              └──────┬───────┘
                                     │                             │
                                     └──────────┬──────────────────┘
                                                ▼
                                         ┌──────────────┐
                                         │   Grafana    │
                                         │  (dashboard) │
                                         └──────────────┘
```

## Prerequisites

The following operators must be installed in the cluster **before** deploying this chart:


helm repo add strimzi https://strimzi.io/charts/
helm install strimzi-operator strimzi/strimzi-kafka-operator

| Operator/Provisioner | Purpose | Install |
|---|---|---|
| [Strimzi Kafka Operator](https://strimzi.io/) | Manages Kafka CRDs | `helm install strimzi oci://quay.io/strimzi-helm/strimzi-kafka-operator` |
| [OpenTelemetry Operator](https://opentelemetry.io/docs/kubernetes/operator/) | Manages OTel Collector CRDs | `helm install otel-operator open-telemetry/opentelemetry-operator --set admissionWebhooks.certManager.enabled=false --set admissionWebhooks.autoGenerateCert.enabled=true` |
| [Rancher Local Path Provisioner](https://github.com/rancher/local-path-provisioner) | Provides `local-path` StorageClass for Kubeadm | `kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml` |

## Chart Structure

```
dofus-observability/
├── Chart.yaml                           # Chart metadata & sub-chart dependencies
├── Chart.lock                           # Pinned dependency versions
├── values.yaml                          # Helm values for all sub-charts & components
├── README.md                            # Chart documentation
├── charts/                              # Downloaded Helm dependency charts (.tgz)
├── configs/                             # Standalone configuration files
│   ├── bot-alerts.yaml                  # Loki log-based alerting rules
│   └── grafana-dashboards/              # Grafana dashboard JSON definitions
│       ├── dofus-dashboard.json         # Dofus Bot Fleet dashboard (Kamas balance + bot logs)
│       └── node-dashboard.json          # Node infrastructure dashboard
└── templates/                           # Kubernetes manifests & Helm templates
    ├── argocd.yaml                      # Ingress route for ArgoCD UI (argocd.dofus.local)
    ├── kafka-cluster.yaml               # Strimzi Kafka Cluster + KafkaNodePool + KafkaTopic + KafkaBridge
    ├── loki-alerts-cm.yaml              # ConfigMap injecting bot-alerts.yaml into Loki
    └── metallb-config.yaml              # MetalLB IPAddressPool + L2Advertisement
```

## Design Principles

### Dashboards as Code & Separation of Concerns

All tool configurations live as **standalone files** in the `configs/` directory:

- **No inline YAML/JSON** inside Kubernetes manifests
- Templates use `{{ .Files.Get "configs/..." }}` to inject configs at deploy time
- Grafana dashboards are loaded via the **sidecar pattern** (ConfigMap label `grafana_dashboard: "1"`)
- Alerting rules are loaded via the **PrometheusRule CRD** with auto-discovery labels

## Installation

### 1. Add Helm Repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 2. Build Dependencies

```bash
cd dofus-observability
helm dependency build
```

### 3. Deploy

```bash
helm install dofus-observability . \
  --namespace observability \
  --create-namespace \
  --values values.yaml
```

### 4. Upgrade

```bash
helm upgrade dofus-observability . \
  --namespace observability \
  --values values.yaml
```

## Key Configuration

### Kafka

| Parameter | Default | Description |
|---|---|---|
| `kafka.replicas` | `3` | Number of Kafka broker replicas |
| `kafka.storage.size` | `50Gi` | Persistent storage per broker |
| `kafka.storage.storageClass` | `local-path` | Storage class for PVCs |
| `kafka.topic.partitions` | `6` | Partitions for `bot_events` topic |

### OpenTelemetry Collector

| Parameter | Default | Description |
|---|---|---|
| `otelCollector.replicas` | `2` | Number of collector replicas |
| `otelCollector.image` | `otel/opentelemetry-collector-contrib:0.115.0` | Collector image |
| `otelCollector.mode` | `deployment` | Deployment mode |

### Alerting

Log-based alerts configured in [`configs/bot-alerts.yaml`](configs/bot-alerts.yaml) and deployed via ConfigMap:

| Alert | Condition / Expression | Severity | Status |
|---|---|---|---|
| `BannedAccountDetected` | `bannedAccountsCount > 0` in last 30m | 🔴 `critical` | Active |
| `ZeroLogIngestion` | No logs from OTLP exporter for 20m | 🔴 `critical` | Active |
| `ZeroControllerLogIngestion` | No logs from Controller bots for 20m | 🔴 `critical` | Active |
| `ZeroBotConnected` | `connectedAccountsCount == 0` for 20m | 🟡 `warning` | Active |
| `TotalKamasDrop` | Net kamas loss > 10 over 1h | 🟡 `warning` | ⚪ *Disabled* |

## Grafana Dashboard

The **Dofus Bot Fleet** dashboard (`uid: dofus-bots`) contains:

1. **Kamas Balance Over Time** — Time Series panel tracking `dofus_bot_kamas` per bot/server
2. **Bot Events Log Stream** — Logs panel querying Loki for `{topic="bot_events"}`

Template variables: `bot_name`, `server` (multi-select with "All" option)

## Uninstall

```bash
helm uninstall dofus-observability --namespace observability
```

## License

Internal use only — DevOps Team.
