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
├── Chart.yaml                           # Dependencies: kube-prometheus-stack, loki-stack
├── values.yaml                          # All configurable values
├── configs/                             # Standalone config files (separation of concerns)
│   ├── otel-collector-config.yaml       # OTel pipeline: Kafka → Prometheus + Loki
│   ├── prometheus-alerts.yaml           # Alert: KamasSuddenDrop (>500k in 5m)
│   ├── grafana-datasources.yaml         # Grafana provisioning: Prometheus + Loki
│   └── dofus-dashboard.json             # Grafana dashboard: Kamas graph + Logs panel
├── templates/                           # Kubernetes manifests (no inline configs!)
│   ├── kafka-cluster.yaml               # Strimzi Kafka + KafkaTopic (bot_events)
│   ├── otel-collector-crd.yaml          # OpenTelemetryCollector CRD
│   ├── prometheus-rule.yaml             # PrometheusRule CRD
│   └── grafana-dashboard-cm.yaml        # ConfigMap with grafana_dashboard label
└── README.md
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

| Alert | Condition | Severity |
|---|---|---|
| `KamasSuddenDrop` | `dofus_bot_kamas` drops > 500k in 5 minutes | `critical` |

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
