# Monitoring RKE2 Clusters

This guide covers setting up monitoring for RKE2 clusters using Prometheus and Grafana.

## Table of Contents

- [Overview](#overview)
- [Quick Start with kube-prometheus-stack](#quick-start-with-kube-prometheus-stack)
- [Manual Installation](#manual-installation)
- [Key Metrics to Monitor](#key-metrics-to-monitor)
- [Alerting Rules](#alerting-rules)
- [Grafana Dashboards](#grafana-dashboards)

## Overview

RKE2 exposes metrics in Prometheus format from:
- **kube-apiserver**: API server metrics (port 6443)
- **kubelet**: Node and container metrics (port 10250)
- **etcd**: etcd cluster metrics (port 2379)
- **kube-scheduler**: Scheduling metrics
- **kube-controller-manager**: Controller metrics

## Quick Start with kube-prometheus-stack

The easiest way to set up monitoring is using the kube-prometheus-stack Helm chart.

### Prerequisites

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add Prometheus community repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Deploy Monitoring Stack

Create values file:

```yaml
# monitoring-values.yaml
prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

grafana:
  adminPassword: "your-secure-password"
  persistence:
    enabled: true
    size: 10Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

# RKE2-specific configuration
kubeEtcd:
  enabled: true
  endpoints:
    - 10.0.0.10  # Your server IPs
    - 10.0.0.11
    - 10.0.0.12
  service:
    enabled: true
    port: 2381
    targetPort: 2381

kubeControllerManager:
  enabled: true
  endpoints:
    - 10.0.0.10
    - 10.0.0.11
    - 10.0.0.12

kubeScheduler:
  enabled: true
  endpoints:
    - 10.0.0.10
    - 10.0.0.11
    - 10.0.0.12

kubeProxy:
  enabled: true
```

Install:

```bash
kubectl create namespace monitoring

helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f monitoring-values.yaml
```

### Access Grafana

```bash
# Port-forward to access locally
kubectl port-forward -n monitoring svc/prometheus-stack-grafana 3000:80

# Or expose via Ingress
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
spec:
  rules:
    - host: grafana.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-stack-grafana
                port:
                  number: 80
EOF
```

## Manual Installation

### Deploy Using Ansible Manifests

Add monitoring manifests to your deployment:

```yaml
# inventory/group_vars/all.yml
rke2_manifest_config_post_run_directory: "./manifests/post-run"
```

Create the manifest directory:

```bash
mkdir -p manifests/post-run
```

### Prometheus ConfigMap

```yaml
# manifests/post-run/prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
          - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)

      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
```

## Key Metrics to Monitor

### Cluster Health

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `up` | Target availability | == 0 |
| `kube_node_status_condition` | Node conditions | Ready != True |
| `etcd_server_has_leader` | etcd leader status | == 0 |

### API Server

```promql
# API server request latency
histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb))

# API server error rate
sum(rate(apiserver_request_total{code=~"5.."}[5m])) / sum(rate(apiserver_request_total[5m]))

# API server request rate
sum(rate(apiserver_request_total[5m])) by (verb)
```

### etcd

```promql
# etcd leader changes
changes(etcd_server_leader_changes_seen_total[1h])

# etcd disk sync duration
histogram_quantile(0.99, sum(rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) by (le))

# etcd database size
etcd_mvcc_db_total_size_in_bytes
```

### Node Resources

```promql
# CPU usage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage
(1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100
```

### Pod Resources

```promql
# Pod CPU usage
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod, namespace)

# Pod memory usage
sum(container_memory_working_set_bytes) by (pod, namespace)

# Pod restart count
sum(kube_pod_container_status_restarts_total) by (pod, namespace)
```

## Alerting Rules

### Critical Alerts

```yaml
# manifests/post-run/prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: rke2-cluster-alerts
  namespace: monitoring
spec:
  groups:
    - name: rke2.rules
      rules:
        - alert: NodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} is not ready"
            description: "Node has been in NotReady state for more than 5 minutes"

        - alert: EtcdNoLeader
          expr: etcd_server_has_leader == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "etcd cluster has no leader"
            description: "etcd cluster has lost its leader"

        - alert: EtcdHighNumberOfLeaderChanges
          expr: increase(etcd_server_leader_changes_seen_total[1h]) > 3
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "etcd leader changes detected"
            description: "etcd has seen {{ $value }} leader changes in the last hour"

        - alert: KubeAPIServerDown
          expr: absent(up{job="kubernetes-apiservers"} == 1)
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Kubernetes API server is down"

        - alert: HighNodeCPU
          expr: (100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 90
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High CPU usage on {{ $labels.instance }}"
            description: "CPU usage is above 90% for more than 10 minutes"

        - alert: HighNodeMemory
          expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage on {{ $labels.instance }}"

        - alert: NodeDiskPressure
          expr: (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100 > 85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Disk usage high on {{ $labels.instance }}"

        - alert: PodCrashLooping
          expr: increase(kube_pod_container_status_restarts_total[1h]) > 5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"

        - alert: EtcdBackupMissing
          expr: time() - rke2_etcd_snapshot_last_success_timestamp > 86400
          for: 1h
          labels:
            severity: warning
          annotations:
            summary: "etcd backup not taken in 24 hours"
```

## Grafana Dashboards

### Import Pre-built Dashboards

Recommended dashboard IDs from Grafana.com:

| Dashboard | ID | Description |
|-----------|-----|-------------|
| Kubernetes Cluster | 7249 | Overall cluster health |
| Node Exporter | 1860 | Node-level metrics |
| etcd | 3070 | etcd cluster metrics |
| CoreDNS | 5926 | DNS metrics |
| Kubernetes Pods | 6336 | Pod-level metrics |

### Import via Grafana UI

1. Go to Dashboards → Import
2. Enter dashboard ID
3. Select Prometheus data source
4. Click Import

### Custom RKE2 Dashboard JSON

```json
{
  "dashboard": {
    "title": "RKE2 Cluster Overview",
    "panels": [
      {
        "title": "Cluster Nodes",
        "type": "stat",
        "targets": [
          {
            "expr": "count(kube_node_info)",
            "legendFormat": "Total Nodes"
          }
        ]
      },
      {
        "title": "Ready Nodes",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(kube_node_status_condition{condition=\"Ready\",status=\"true\"})",
            "legendFormat": "Ready"
          }
        ]
      },
      {
        "title": "etcd Members",
        "type": "stat",
        "targets": [
          {
            "expr": "count(etcd_server_has_leader)",
            "legendFormat": "Members"
          }
        ]
      },
      {
        "title": "API Server Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(apiserver_request_total[5m])) by (verb)",
            "legendFormat": "{{ verb }}"
          }
        ]
      }
    ]
  }
}
```

## Alertmanager Configuration

Configure notifications:

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-prometheus-stack-kube-prom-alertmanager
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m

    route:
      group_by: ['alertname', 'severity']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'default'
      routes:
        - match:
            severity: critical
          receiver: 'critical'

    receivers:
      - name: 'default'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/xxx'
            channel: '#alerts'

      - name: 'critical'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/xxx'
            channel: '#critical-alerts'
        pagerduty_configs:
          - service_key: 'your-pagerduty-key'
```
