---
title: "Log based alerting with Loki"
date: 2025-12-27
author: Yadav Lamichhane
description: "Enable log-based alerting with Grafana/Loki"
tags:
  - k8s
---

# Log-Based Alerting with Loki on Kubernetes

This repository documents how to deploy and operate **log-based alerting in Kubernetes using Grafana Loki** as part of a complete observability stack.

Instead of relying only on metrics, this setup enables **alerts driven directly from application and system logs** using Loki Ruler and LogQL. This allows detection of failures, error patterns, and operational issues that are often invisible to metrics-based monitoring.

---

## Overview

The stack consists of the following components:

- **Prometheus** – Metrics collection and storage
- **Alertmanager** – Alert routing and notification handling
- **Loki** – Log aggregation and indexing
- **Promtail** – Log collection agent for Kubernetes pods
- **Grafana** – Visualization and alert inspection

All components are deployed in the `monitoring` namespace.

---

## Architecture (Logical Flow)

```
Kubernetes Pods
      │
      ▼
  Promtail (DaemonSet)
      │
      ▼
      Loki (Single Binary)
      │
      ├── Log Storage (filesystem)
      ├── LogQL Queries
      └── Loki Ruler
              │
              ▼
        Alertmanager
              │
              ▼
           Slack
```

---

## Prerequisites

- Kubernetes cluster (v1.24+ recommended)
- Helm v3
- `kubectl` configured to access the cluster

---

## Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

---

## Install Prometheus Stack

We use kube-prometheus-stack, which bundles Prometheus, Alertmanager, node exporters, and kube-state-metrics.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.enabled=false
```

---

This installs: - Prometheus Server - Alertmanager - Node Exporter - kube-state-metrics

## Install Promtail

Promtail runs as a DaemonSet and ships Kubernetes pod logs to Loki.

Use the below promtail values `promtail-values.yaml`

```yaml
daemonset:
  enabled: true

deployment:
  enabled: false

config:
  serverPort: 3101

  clients:
    - url: http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push

  positions:
    filename: /run/promtail/positions.yaml

  scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
        - role: pod

      pipeline_stages:
        - docker: {}

      relabel_configs:
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace

        - source_labels: [__meta_kubernetes_pod_name]
          target_label: pod

        - source_labels: [__meta_kubernetes_pod_container_name]
          target_label: container

        - source_labels:
            [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
          separator: /
          target_label: __path__
          replacement: /var/log/pods/*$1/*.log
```

```bash
helm install promtail grafana/promtail \
  -n monitoring \
  -f promtail-values.yaml
```

---

## Install Loki (Single Binary Mode)

Loki is deployed in **SingleBinary mode** to keep the setup simple. All distributed components are disabled.

```bash
helm install loki -n monitoring grafana/loki -f values.yaml
```

Where `values.yaml` contains:

```yaml
deploymentMode: SingleBinary

backend:
  replicas: 0
read:
  replicas: 0
write:
  replicas: 0

ingester:
  replicas: 0
querier:
  replicas: 0
queryFrontend:
  replicas: 0
distributor:
  replicas: 0

chunksCache:
  enabled: false
resultsCache:
  enabled: false
indexGateway:
  enabled: false
gateway:
  enabled: false

monitoring:
  selfMonitoring:
    enabled: false
  lokiCanary:
    enabled: false

singleBinary:
  replicas: 1
  persistence:
    enabled: true
    size: 5Gi

extraVolumes:
  - name: loki-rules
    configMap:
      name: loki-rules

extraVolumeMounts:
  - name: loki-rules
    mountPath: /loki/rules
    readOnly: true

loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1

  structuredConfig:
    ruler:
      rule_path: /tmp/loki/rules-temp
      storage:
        type: local
        local:
          directory: /loki/rules
      alertmanager_url: http://prometheus-kube-prometheus-alertmanager.monitoring.svc:9093
      enable_alertmanager_v2: true
      enable_api: false
      ring:
        kvstore:
          store: inmemory

  storage:
    type: filesystem
    bucketNames:
      chunks: chunks
      ruler: ruler
      admin: admin

  schemaConfig:
    configs:
      - from: 2024-01-01
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
```

This configuration deploys Grafana Loki in Single Binary mode, running ingestion, storage, querying, and alert evaluation in a single pod for simplicity. Distributed Loki components are explicitly disabled to reduce operational overhead. Logs are stored locally using the filesystem backend, and the Loki Ruler is enabled to evaluate LogQL-based alert rules directly against log streams. Generated alerts are sent to Alertmanager, with optional remote write support to Prometheus for correlating log-based alerts with metrics.

Since we are using the loki as a SingleBinary deployment mode, we need to mount the configmap for Loki Alerting rules.yaml explicitly to the pod. Here are the contains of the configmap.

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-rules
  namespace: monitoring
data:
  rules.yaml: |
    groups:
      - name: loki.test
        interval: 1m
        rules:
          - alert: LokiLogDetected
            expr: |
              count_over_time({namespace="monitoring", app="loki"} |= "Response received from loki" [1m]) > 0
            for: 30s
            labels:
              severity: critical
              namespace: monitoring
            annotations:
              summary: "Loki logs detected"
              description: "At least one log line seen from Loki in last minute"
```

Now, apply the configMap

```bash
kubectl apply -f configmap.yaml
```

During the loki, installation, we have defined the extraVolumes and extraVolumeMounts with the configuration from the configmap above.

```bash
...
  extraVolumes:
    - name: loki-rules
      configMap:
        name: loki-rules

  extraVolumeMounts:
    - name: loki-rules
      mountPath: /loki/rules
      readOnly: true
...
```

Now, we need to restart the loki-0 pod, to make the rules.yaml available in mounted path. This step is important for the ruler to function as Ruler read rule files from /loki/rules inside the container.

```bash
...
structuredConfig:
  ruler:
    storage:
      type: local
      local:
        directory: /loki/rules
...
```

Here is the output how the rules.yaml is mounted in the pod.

```bash
❯ kubectl exec -it -n monitoring loki-0 -- cat /loki/rules/rules.yaml
E1227 12:12:19.265613   65721 websocket.go:296] Unknown stream id 1, discarding message
groups:
  - name: loki.test
    interval: 1m
    rules:
      - alert: LokiLogDetected
        expr: |
          count_over_time({namespace="monitoring", app="loki"} |= "Response received from loki" [1m]) > 0
        for: 30s
        labels:
          severity: critical
          namespace: monitoring
        annotations:
          summary: "Loki logs detected"
          description: "At least one log line seen from Loki in last minute"
```

### Configure Grafana Data Sources

Create a ConfigMap to automatically provision Prometheus, Loki, and Alertmanager as Grafana data sources.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-operated.monitoring.svc.cluster.local:9090
        isDefault: true
      - name: Loki
        type: loki
        url: http://loki.monitoring.svc.cluster.local:3100
      - name: Alertmanager
        type: alertmanager
        url: http://prometheus-kube-prometheus-alertmanager.monitoring.svc.cluster.local:9093
```

Apply the ConfigMap:

```bash
kubectl apply -f grafana-datasources.yaml
```

### Install Grafana

Grafana is deployed with persistence and the above data sources mounted.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install grafana grafana/grafana \
  --namespace monitoring \
  --set adminPassword=admin \
  --set persistence.enabled=true \
  --set service.type=NodePort \
  --set extraConfigmapMounts[0].name=grafana-datasources \
  --set extraConfigmapMounts[0].mountPath=/etc/grafana/provisioning/datasources \
  --set extraConfigmapMounts[0].configMap=grafana-datasources \
  --set extraConfigmapMounts[0].readOnly=true
```

Configure Alertmanager Notifications
Alertmanager supports email, slack, pagerduty, webhooks, etc., in this post we will configure the slack. For configuration, fetch the default alertmanager config from the pod where we will be adding the slack configuration.

```bash
kubectl -n monitoring get secret alertmanager-prometheus-kube-prometheus-alertmanager \
  -o jsonpath="{.data.alertmanager\.yaml}" | base64 --decode > alertmanager.yaml
```

### Create Slack Webhook

    1. Go to Slack API Apps
    2. Create new app → “From scratch”
    3. Add Incoming Webhooks feature
    4. Copy webhook URL (format: https://hooks.slack.com/services/T000/B000/XXXX)

Update the alertmanager configuration file alertmanager.yaml

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: "https://hooks.slack.com/services/T000/B000/XXXX" # Replace with your Slack webhook URL

route:
  group_wait: 5s
  group_interval: 30s
  repeat_interval: 1m

  receiver: "null"

  routes:
    - receiver: slack
      matchers:
        - severity =~ "critical|warning|info"
      continue: true

receivers:
  - name: "null"

  - name: slack
    slack_configs:
      - send_resolved: true
        channel: "#alert"
        title: "{{ .CommonLabels.alertname }}"
        text: |-
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Namespace:* {{ .Labels.namespace }}
          *Starts At:* {{ .StartsAt.Format "2006-01-02 15:04:05 UTC" }}
          ---
          {{ end }}

inhibit_rules:
  - target_matchers:
      - severity =~ warning|info
    source_matchers:
      - severity = critical
    equal:
      - namespace
      - alertname

  - target_matchers:
      - severity = info
    source_matchers:
      - severity = warning
    equal:
      - namespace
      - alertname

  - target_matchers:
      - severity = info
    source_matchers:
      - alertname = InfoInhibitor
    equal:
      - namespace

  - target_matchers:
      - alertname = InfoInhibitor

templates:
  - /etc/alertmanager/config/*.tmpl
```

Apply the updated configuration back to Alertmanager

```bash
# Encode and patch alertmanager secret
kubectl -n monitoring patch secret alertmanager-prometheus-kube-prometheus-alertmanager --type merge -p "{\"data\":{\"alertmanager.yaml\":\"$( base64 -w0 < alertmanager.yaml )\"}}"
```

Restart the alertmanager component

```bash
kubectl -n monitoring rollout restart statefulset \
  alertmanager-prometheus-kube-prometheus-alertmanager
```

## Access Grafana

```bash
kubectl get svc -n monitoring grafana
```

Access Grafana in browser:

```bash
http://<NodeIP>:<NodePort>
```

Default credentials:

    - Username: admin
    - Password: admin

### Logs with Loki

Examine the query that we have set in the rules.yaml to validate if query is resulting into any value.

```logql

count_over_time({namespace="monitoring", app="loki"} |= "Response received from loki" [1m]) > 0

```

![Logs in Grafana Dashboard](https://imgur.com/QmL05xo.png)

You can correlate logs directly with metrics inside Grafana. With this setup, now you can check the alertmanager UI to visualize if rules.yaml is converted into the alerts.

If everything is valid, we should be having the alerts routed to the alertmanager via ruler configuration, to validate if alerts are present in the alertmanager we need to port-forward the alertmanager service to port 9093.

```bash
kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-alertmanager 9093
```

Access the Alertmanager UI from the browser `http://localhost:9093`

![Alerts in Alertmanager](https://imgur.com/eM1QltZ.png)

Here we can see, alertname name exactly matches the content of the rules.yaml configured so we should be having alert in the slack as well.

![Slack Alerts](https://imgur.com/09d5kBF.png)

Finally, logs based alerts landed in the slack. Now we can add as many rules to enable the log based alerts and receive them in the slack or any alerting tools you may like.
