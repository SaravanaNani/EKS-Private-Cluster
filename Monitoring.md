# EKS Monitoring Stack — Prometheus, Grafana, Loki & Promtail

A complete, reproducible setup for metrics and logs across **Amazon EKS** (Prometheus + exporters + Promtail) and a **bastion/monitoring server** (Grafana + Loki).

---

## Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Setup — EKS Side](#setup--eks-side)
  - [Namespaces](#namespaces)
  - [Prometheus Stack](#prometheus-stack)
  - [Sample App (adq-dev/devops-task-app)](#sample-app-adq-devdevops-task-app)
  - [Promtail DaemonSet](#promtail-daemonset)
- [Setup — Bastion/Monitoring Server](#setup--bastionmonitoring-server)
  - [Loki](#loki)
  - [Grafana](#grafana)
- [Ingress / Access](#ingress--access)
- [Validation](#validation)
- [Troubleshooting & Fixes](#troubleshooting--fixes)
- [Final Architecture Diagram](#final-architecture-diagram)
- [Appendix — Useful Commands](#appendix--useful-commands)

---

## Project Overview

This project delivers a **production-style observability stack**:

- **Metrics**: Prometheus scrapes Kubernetes, node, and container metrics (via **kube-state-metrics**, **node-exporter**, **cAdvisor**).
- **Logs**: **Promtail** runs as a DaemonSet on EKS nodes, scraping
  - host/system logs: `/var/log/**/*.log`
  - Kubernetes pod/container logs: `/var/log/pods/.../*.log`
  - application file logs mounted on host: `/var/log/adq-dev/*.log`
  and **pushes** them to **Loki** running on the **bastion**.
- **Dashboards**: **Grafana** (on bastion) uses Loki datasource for logs and Prometheus datasource (optional) for metrics.

Why this approach?
- Clear separation: EKS focuses on collection; bastion focuses on storage/visualization.
- EKS-friendly: works with **containerd** (default on EKS).
- Scales: add more EKS nodes (Promtail follows), scale Loki/Grafana as needed.

---

## Architecture

**Distribution**

- **EKS cluster**
  - Prometheus server and exporters (node-exporter, cAdvisor, kube-state-metrics)
  - Promtail (DaemonSet)
  - Sample application (`adq-dev/devops-task-app`)

- **Bastion/Monitoring server**
  - Loki (single binary)
  - Grafana (Loki datasource)

**Flow**

1. App logs → stdout & file → kubelet writes to `/var/log/pods` (and `/var/log/containers` symlinks).
2. Promtail (on every node) tails system logs, pod logs, and custom app file logs.
3. Promtail **pushes** logs to Loki `http://<bastion-ip>:3100`.
4. Grafana queries Loki for dashboards and ad‑hoc searches.

---

## Prerequisites

- An EKS cluster (kubectl context set).
- A bastion/monitoring Linux host reachable from EKS node subnets on TCP **3100** (Loki) and accessible from your browser on **3000** (Grafana).
- `kubectl`, `jq`, Docker (for building the sample app image), and permissions to create cluster resources.
- Open security groups/NACLs accordingly (EKS nodes → bastion:3100).

---

## Setup — EKS Side

### Namespaces

```bash
kubectl create ns monitoring || true
kubectl create ns adq-dev || true
```

### Prometheus Stack

> Use your preferred method (Prometheus Operator/Helm). Ensure the following are running in **monitoring**:
- `prometheus-prometheus-0`
- `kube-state-metrics`
- `node-exporter` (DaemonSet)
- `cadvisor` (DaemonSet)

Validate:
```bash
kubectl -n monitoring get pods
```

### Sample App (`adq-dev/devops-task-app`)

**Dockerfile**
```dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app
COPY . .
RUN npm install express
EXPOSE 3000
CMD ["node", "app.js"]
```

**app.js**
```js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  console.log(`[${new Date().toISOString()}] GET / request received`);
  res.send('🚀 Sample Node App for Promtail Logging Demo!');
});

setInterval(() => {
  console.log(`[${new Date().toISOString()}] App heartbeat log`);
}, 5000);

app.listen(3000, () => console.log('Server running on port 3000'));
```

**Kubernetes Deployment** (writes to stdout **and** host-file; Promtail scrapes both)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-task-app
  namespace: adq-dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devops-task-app
  template:
    metadata:
      labels:
        app: devops-task-app
    spec:
      containers:
      - name: devops-task-app
        image: saravana2002/demo-node-logger:latest
        ports:
        - containerPort: 3000
        command: ["/bin/sh","-c"]
        args:
          - |
            mkdir -p /usr/src/app/logs && \
            node app.js 2>&1 | tee -a /usr/src/app/logs/app.log
        volumeMounts:
        - name: app-logs
          mountPath: /usr/src/app/logs
      volumes:
      - name: app-logs
        hostPath:
          path: /var/log/adq-dev
          type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  name: devops-task-app
  namespace: adq-dev
spec:
  selector:
    app: devops-task-app
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

Apply:
```bash
kubectl -n adq-dev apply -f devops-task-app.yaml
kubectl -n adq-dev get pods -o wide
```

### Promtail DaemonSet

**Final working promtail.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: promtail
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: promtail
rules:
  - apiGroups: [""]
    resources: ["pods","nodes","namespaces"]
    verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: promtail
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: promtail
subjects:
  - kind: ServiceAccount
    name: promtail
    namespace: monitoring
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: monitoring
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
      grpc_listen_port: 0

    positions:
      filename: /run/promtail/positions.yaml

    clients:
      - url: http://10.10.101.140:3100/loki/api/v1/push

    scrape_configs:
      # 1) System logs
      - job_name: varlogs
        static_configs:
          - targets: [localhost]
            labels:
              job: varlogs
              __path__: /var/log/**/*.log

      # 2) Kubernetes pod logs (ALL namespaces)
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        pipeline_stages:
          - docker: {}
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_container_name]
            target_label: container
          - source_labels: [__meta_kubernetes_node_name]
            target_label: node
          - action: replace
            target_label: job
            replacement: kubernetes-pods
          - source_labels:
              [__meta_kubernetes_namespace, __meta_kubernetes_pod_name, __meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
            separator: "_"
            target_label: __path__
            replacement: /var/log/pods/$1_$2_$3/$4/*.log

      # 3) Application file logs
      - job_name: app-logs
        static_configs:
          - targets: [localhost]
            labels:
              job: app-logs
              namespace: adq-dev
              __path__: /var/log/adq-dev/*.log
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: monitoring
  labels: { app: promtail }
spec:
  selector:
    matchLabels: { app: promtail }
  template:
    metadata:
      labels: { app: promtail }
    spec:
      serviceAccountName: promtail
      hostNetwork: true
      tolerations:
        - operator: Exists
      containers:
        - name: promtail
          image: grafana/promtail:2.9.0
          args:
            - -config.file=/etc/promtail/promtail.yaml
            - -config.expand-env=true
          env:
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: config
              mountPath: /etc/promtail
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: podlogs
              mountPath: /var/log/pods
              readOnly: true
            - name: positions
              mountPath: /run/promtail
      volumes:
        - name: config
          configMap:
            name: promtail-config
            items:
              - key: promtail.yaml
                path: promtail.yaml
        - name: varlog
          hostPath:
            path: /var/log
        - name: podlogs
          hostPath:
            path: /var/log/pods
            type: DirectoryOrCreate
        - name: positions
          hostPath:
            path: /run/promtail
            type: DirectoryOrCreate
```

Apply:
```bash
kubectl -n monitoring apply -f promtail.yaml
kubectl -n monitoring rollout status ds/promtail
kubectl -n monitoring get pods -l app=promtail -o wide
```

---

## Setup — Bastion/Monitoring Server

### Loki

**/etc/loki/loki-config.yaml** (compatible with Loki v2.8.2 used here)

```yaml
server:
  http_listen_port: 3100
  http_listen_address: 0.0.0.0
  log_level: info

auth_enabled: false

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  wal:
    dir: /var/loki/wal

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /var/loki/index
    cache_location: /var/loki/cache
    shared_store: filesystem
  filesystem:
    directory: /var/loki/chunks

compactor:
  working_directory: /var/loki/compactor

limits_config:
  ingestion_rate_mb: 8
  ingestion_burst_size_mb: 16
  max_entries_limit_per_query: 5000
  max_streams_per_user: 10000
  reject_old_samples: true
```

Run foreground (for tests):
```bash
/usr/local/bin/loki -config.file=/etc/loki/loki-config.yaml
```

Or as a systemd service `/etc/systemd/system/loki.service`:
```ini
[Unit]
Description=Loki Log Aggregator
After=network.target

[Service]
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yaml
Restart=always
User=loki
Group=loki
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now loki
sudo systemctl status loki
```

### Grafana

- Install Grafana on bastion.
- Add **Loki datasource**: `http://10.10.101.140:3100` (adjust IP).
- Build dashboards for:
  - Logs by namespace/job
  - Live tail for `app-logs` in `adq-dev`
  - Error rates using LogQL filters

> **LogQL note**: use **non-empty** matchers to avoid datasource errors.  
> ✅ `{job!="", namespace!=""}` or `{job=~".+", namespace=~".+"}`  
> ❌ `{job=~".*"}`

---

## Ingress / Access

- **Grafana**: `http://<bastion-ip>:3000` (datasource → Loki).
- **Loki**: `http://<bastion-ip>:3100` (promtail clients).
- **Prometheus**: via port-forward or your ALB/Ingress:
  ```bash
  kubectl -n monitoring port-forward svc/prometheus-operated 9090:9090
  # open http://localhost:9090/targets
  ```

Ensure security groups allow EKS nodes → bastion:3100 (Loki).

---

## Validation

**Promtail tailing files**
```bash
kubectl -n monitoring logs -l app=promtail --tail=200 \
  | grep -E "tail routine|Seeked" | head -40
```

**Check visible pod/container logs inside Promtail**
```bash
PROMTAIL=$(kubectl -n monitoring get pod -l app=promtail -o jsonpath='{.items[0].metadata.name}')
kubectl -n monitoring exec -it "$PROMTAIL" -- sh -c '
  echo ">>> /var/log/containers"
  ls -l /var/log/containers | head -10
  echo ">>> /var/log/pods"
  find /var/log/pods -maxdepth 3 -type f -name "*.log" | head -10
'
```

**Loki API checks**
```bash
LOKI="http://10.10.101.140:3100"

curl -s "$LOKI/loki/api/v1/labels" | jq .
curl -s "$LOKI/loki/api/v1/label/job/values" | jq .
curl -s "$LOKI/loki/api/v1/label/namespace/values" | jq .

# Instant
for j in varlogs kubernetes-pods app-logs; do
  echo -n "$j: "
  curl -G -s "$LOKI/loki/api/v1/query" \
    --data-urlencode "query={job=\"$j\"}" | jq '.data.result | length'
done

# Range (last 5m)
START=$(($(date +%s)-300))000000000; END=$(date +%s)000000000
curl -G -s "$LOKI/loki/api/v1/query_range" \
  --data-urlencode 'query={job!="",namespace!=""}' \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode 'limit=5' | jq '.data.result | length'
```

**Grafana**
- Explore → Query `{job="app-logs", namespace="adq-dev"} |= ""`
- Build panels with LogQL using non-empty matchers.

---

## Troubleshooting & Fixes

- **Node app crash**: container tried to start `server.js` (missing).  
  _Fix_: Start `app.js` (Dockerfile & CMD).

- **Promtail didn’t see pod logs**: missing `/var/log/pods` mount and `__path__` mapping.  
  _Fix_: Add hostPath mounts and relabel to `__path__ = /var/log/pods/$ns_$pod_$uid/$container/*.log`.

- **Docker container logs empty on EKS**: EKS uses **containerd**; Docker path `/var/lib/docker/containers/*/*.log` often unused.  
  _Fix_: rely on **kubernetes-pods** job.

- **Grafana “empty-compatible matcher” error**: `{job=~".*"}` is invalid.  
  _Fix_: use `{job=~".+"}` or `{job!=""}`.

- **Loki “too many outstanding requests” & parse errors**:  
  _Fix_: add `limits_config`; ensure Loki config matches **v2.8.2** (remove unsupported keys like `retention_period`, `allow_structured_metadata`, `reload_period`).

- **Positions file warnings** after rotation: benign; Promtail catches up.  
  _Fix (optional)_: delete `/run/promtail/positions.yaml` in DS pod; it’ll rebuild.

---

## Final Architecture Diagram

```
Bastion / Monitoring
┌───────────────────────────────────────────────────────────────┐
│ Grafana (:3000) ──(Loki datasource)─────┐                    │
│                                         │                    │
│ Loki (:3100) ◀──── Promtail (push) ◀────┘                    │
└───────────────────────────────────────────────────────────────┘

Amazon EKS
┌───────────────────────────────────────────────────────────────┐
│ Prometheus stack:                                             │
│  • prometheus-prometheus-0                                    │
│  • kube-state-metrics                                         │
│  • node-exporter (DS)                                         │
│  • cAdvisor (DS)                                              │
│                                                               │
│ Promtail (DS) — scrapes:                                      │
│  • /var/log/**/*.log (system)                                 │
│  • /var/log/pods/.../*.log (k8s containers)                   │
│  • /var/log/adq-dev/*.log (app file logs)                     │
│  → pushes to Loki on bastion                                  │
│                                                               │
│ Sample app: adq-dev/devops-task-app (Node.js)                 │
│  • stdout + file logs                                         │
└───────────────────────────────────────────────────────────────┘
```

---

## Appendix — Useful Commands

```bash
# restart/review promtail
kubectl -n monitoring rollout restart ds/promtail
kubectl -n monitoring logs -l app=promtail --tail=200

# which node runs a pod? which promtail is on that node?
kubectl -n adq-dev get pods -o wide | grep devops-task-app
NODE=$(kubectl -n adq-dev get pod -o wide | awk '/devops-task-app/{print $7; exit}')
kubectl -n monitoring get pod -o wide | awk -v n="$NODE" '$7==n && /promtail/'

# clean promtail positions (if needed)
PROMTAIL=$(kubectl -n monitoring get pod -l app=promtail -o jsonpath='{.items[0].metadata.name}')
kubectl -n monitoring exec -it "$PROMTAIL" -- rm -f /run/promtail/positions.yaml
kubectl -n monitoring rollout restart ds/promtail
```
