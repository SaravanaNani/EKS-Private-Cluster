# Monitoring on EKS with Loki/Promtail & Grafana — Interview Q&A

## Intro (use in interviews)
I designed and implemented a full observability stack for workloads on Amazon EKS. Metrics are collected with Prometheus and exporters on the cluster, while logs are shipped by Promtail DaemonSets to a centralized Loki instance running on a bastion/monitoring server. Grafana, also on the bastion, visualizes both metrics and logs. I handled several real-world issues (container runtime differences, Loki limits, LogQL matchers, and Kubernetes log path mapping) and documented validation and troubleshooting procedures.

---

## Questions & Answers

### 1) Why did you place Prometheus on EKS but Loki + Grafana on a bastion?
**Answer:** Prometheus scrapes internal k8s targets and benefits from running close to the cluster. Loki and Grafana serve as centralized log storage and visualization; placing them on a bastion simplifies external access and isolates storage and UI from cluster lifecycle. It also lets the cluster scale independently while keeping a stable monitoring endpoint.

### 2) How does Promtail forward logs to Loki?
**Answer:** Promtail runs as a DaemonSet on every node. It tails files from `/var/log/**/*.log`, Kubernetes pod logs from `/var/log/pods/<ns>_<pod>_<uid>/<container>/0.log`, and custom app files under `/var/log/adq-dev/*.log`. It labels each stream (job, namespace, pod, container, node) via `relabel_configs` and pushes logs over HTTP to Loki at `http://<bastion-ip>:3100/loki/api/v1/push`.

### 3) Why didn’t you rely on `/var/lib/docker/containers/*/*.log`?
**Answer:** EKS defaults to **containerd**, not Docker. Container stdout/stderr is already written to `/var/log/pods` with symlinks in `/var/log/containers`. The Docker path can be empty, causing confusion. The `kubernetes-pods` job is runtime-agnostic and works reliably on EKS.

### 4) What issues did you face with Promtail and how did you fix them?
**Answer:** Initially, Promtail didn’t ingest pod logs due to missing hostPath mounts and no `__path__` mapping. We mounted `/var/log` and `/var/log/pods` into the Promtail pod and added relabel rules to map Kubernetes meta labels to the real file path. After this, the DaemonSet tailed pod logs correctly across all namespaces.

### 5) How did you handle the Loki error “too many outstanding requests”?
**Answer:** We added a `limits_config` in Loki (`ingestion_rate_mb`, `ingestion_burst_size_mb`, `max_streams_per_user`, etc.), which provides back-pressure behavior and improves stability under bursts.

### 6) Grafana returned “queries require at least one regexp or equality matcher…” — what caused that?
**Answer:** LogQL rejects *empty-compatible* matchers like `{job=~".*"}`. The fix is to use non-empty matchers such as `{job=~".+"}` or `{job!=""}` (and `{namespace!=""}` if we need both).

### 7) Describe the final Promtail relabeling for Kubernetes pod logs.
**Answer:** We enabled `kubernetes_sd_configs` (role: pod), labeled `namespace`, `pod`, `container`, `node`, and set `job="kubernetes-pods"`. Crucially, we built the `__path__` from meta labels:
```
source_labels: [__meta_kubernetes_namespace,__meta_kubernetes_pod_name,__meta_kubernetes_pod_uid,__meta_kubernetes_pod_container_name]
separator: "_"
target_label: __path__
replacement: /var/log/pods/$1_$2_$3/$4/*.log
```

### 8) How did you validate that logs were reaching Loki?
**Answer:** Used the Loki HTTP API:
- `/loki/api/v1/labels`
- `/loki/api/v1/label/job/values`
- `/loki/api/v1/label/namespace/values`
- Queries like:
  ```bash
  curl -G "$LOKI/loki/api/v1/query" --data-urlencode 'query={job="app-logs"}'
  ```
And range queries with `start`/`end` in nanoseconds. In Grafana, we tailed `{job="app-logs", namespace="adq-dev"}`.

### 9) What caused the Node.js app to crash earlier?
**Answer:** The container CMD referenced `server.js`, which didn’t exist. We rebuilt the image to start `app.js`. The new image periodically logs “App heartbeat log”, which helped validate ingestion.

### 10) Why mount `/var/log/adq-dev` and not just rely on stdout?
**Answer:** We wanted to demonstrate both stdout and **file-based** logging. The app writes to `/usr/src/app/logs/app.log`, mounted to host `/var/log/adq-dev/app.log`, which Promtail scrapes via `app-logs` job with `namespace=adq-dev` label.

### 11) How do you access Prometheus targets?
**Answer:** Port-forward the service or expose via Ingress/ALB. With port-forward:
```
kubectl -n monitoring port-forward svc/prometheus-operated 9090:9090
# Open http://localhost:9090/targets
```

### 12) What security considerations exist for this setup?
**Answer:** Network access from EKS nodes to Loki (TCP 3100) must be allowed. In production, enable auth/mTLS for Loki and Grafana, restrict Promtail client origins, and segment networks. Consider S3 or object storage for Loki chunks and backups.

### 13) How would you scale Loki?
**Answer:** Move from single binary to distributed Loki (separate ingesters, distributors, queriers), use object storage (S3) for chunks and indexes, add query frontends, and tune limits and caching.

### 14) How would you add alerts?
**Answer:** Add Alertmanager and recording/alerting rules in Prometheus, and/or use Grafana alerting for log- or metric-based conditions (e.g., error spikes, pod restarts).

### 15) How did you debug Promtail when pod logs didn’t appear?
**Answer:** Checked Promtail logs for “tail routine: started” lines, exec’d into Promtail to list `/var/log/pods` and `/var/log/containers`, verified that Kubernetes symlinks and real files existed, and confirmed relabel/labels with Loki API. Also ensured RBAC allowed pod discovery and that `positions.yaml` was writable.

### 16) Why set `hostNetwork: true` on Promtail?
**Answer:** It simplifies node hostname/IP discovery and avoids oddities with container networking, though it’s optional. Promtail mainly needs hostPath access to `/var/log` and `/var/log/pods`.

### 17) How do you avoid double-scraping the same logs?
**Answer:** Use either `kubernetes-pods` (preferred) **or** a container-runtime specific job. On EKS (containerd), we rely on `kubernetes-pods` and omit Docker-specific paths to prevent duplicates.

### 18) Best practices for production hardening?
**Answer:**
- Authentication and multi-tenancy in Loki
- Object storage backend (S3) and retention policies
- Resource requests/limits for DS and stateful components
- Dashboards + alerts for ingestion failures and high error rates
- Grafana access control and backups

### 19) How did you ensure your LogQL queries didn’t error out?
**Answer:** Always use non-empty matchers, e.g. `{job!=""}` and `{namespace!=""}`. For broad discovery panels, prefer `count_over_time` with a positive filter and keep limits sane.

### 20) How would you extend this to multiple clusters?
**Answer:** Deploy Promtail DS in each cluster, label by cluster (e.g., `cluster=<name>`), and push to the same Loki (or per-cluster Loki). Add `cluster` as a top-level label in dashboards and queries.
