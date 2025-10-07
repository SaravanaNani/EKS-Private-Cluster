
# Files
    [root@ip-10-10-101-140 ~]# ls
    '\'                         devops-task.yaml            output.bin
     aws                        eks-metric                  prometheus
     aws-controller.yaml        eksctl_Linux_amd64.tar.gz   protect-namespaces.yaml
     awscliv2.zip               kubectl                     tls-devops-task.yaml
     devops-task-ingress.yaml   monitoring

 ###  devops-task.yaml  
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
            image: saravana2002/devops-task:latest
            ports:
            - containerPort: 3000   # 👈 app listens inside pod
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
        - port: 80         # 👈 service port
          targetPort: 3000 # 👈 forwards into pod containerPort
      type: ClusterIP      # 👈 always use ClusterIP when behind ingress


###  tls-devops-task.yaml
    
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: saravana09052002aws@gmail.com
        privateKeySecretRef:
          name: letsencrypt-prod
        solvers:
        - http01:
            ingress:
              class: nginx

### devops-task-ingress.yaml

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: devops-task-ingress
      namespace: adq-dev
      annotations:
        cert-manager.io/cluster-issuer: "letsencrypt-prod"
    spec:
      ingressClassName: nginx
      tls:
      - hosts:
        - app.adqget.bar
        secretName: devops-task-tls  # cert will be stored here
      rules:
      - host: app.adqget.bar
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: devops-task-app
                port:
                  number: 80

# eks-metrics:
    cd eks-metric/
    [root@ip-10-10-101-140 eks-metric]# ls
    cAdvisor.yaml        kube-state-metrics.yaml  loki-linux-amd64.zip.1  prometheus-ingress.yaml  promtail.yaml
    ebs-csi-policy.json  loki-linux-amd64.zip     node-expoter.yaml       prometheus.yaml          storageclass.yaml

### storageclass.yaml

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: gp3
    provisioner: ebs.csi.aws.com   # <-- Use CSI driver instead of in-tree
    parameters:
      type: gp3
    reclaimPolicy: Delete
    volumeBindingMode: WaitForFirstConsumer

 ###  prometheus.yaml 

     ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: monitoring
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: prometheus
      namespace: monitoring
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: prometheus
    rules:
      - apiGroups: [""]
        resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods", "namespaces"]
        verbs: ["get", "list", "watch"]
      - apiGroups: ["apps", "extensions"]
        resources: ["deployments", "replicasets", "daemonsets", "statefulsets"]
        verbs: ["get", "list", "watch"]
      - apiGroups: ["networking.k8s.io"]
        resources: ["ingresses"]
        verbs: ["get", "list", "watch"]
      - apiGroups: ["coordination.k8s.io"]
        resources: ["leases"]
        verbs: ["get", "list", "watch", "create", "update", "patch"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: prometheus
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: prometheus
    subjects:
      - kind: ServiceAccount
        name: prometheus
        namespace: monitoring
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: prometheus-config
      namespace: monitoring
    data:
      prometheus.yml: |
        global:
          scrape_interval: 15s
          evaluation_interval: 30s
    
        scrape_configs:
    
          - job_name: 'node-exporter'
            static_configs:
            - targets: ['node-exporter.monitoring.svc.cluster.local:9100']
    
    
    
          # Node Exporter
          - job_name: "node-exporter"
            kubernetes_sd_configs:
              - role: endpoints
            relabel_configs:
              - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_name]
                regex: node-exporter
                action: keep
    
          # Kube State Metrics
          - job_name: "kube-state-metrics"
            kubernetes_sd_configs:
              - role: endpoints
            relabel_configs:
              - source_labels: [__meta_kubernetes_service_label_app]
                regex: kube-state-metrics
                action: keep
    
          # cAdvisor
          - job_name: "cadvisor"
            kubernetes_sd_configs:
              - role: pod
            relabel_configs:
              - source_labels: [__meta_kubernetes_pod_label_app]
                regex: cadvisor
                action: keep
    
              - source_labels: [__meta_kubernetes_pod_container_port_number]
                regex: "8080"
                action: keep
    
          # Kubernetes Pods (scrape if annotated)
          - job_name: "kubernetes-pods"
            kubernetes_sd_configs:
              - role: pod
            relabel_configs:
              - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
                regex: "true"
                action: keep
              - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
                target_label: __metrics_path__
                regex: (.+)
                action: replace
              - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
                target_label: __address__
                regex: (.+):\d+;(\d+)
                replacement: $1:$2
                action: replace
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: prometheus-data
      namespace: monitoring
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: gp3
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: prometheus
      namespace: monitoring
      labels:
        app: prometheus
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: prometheus
      template:
        metadata:
          labels:
            app: prometheus
        spec:
          serviceAccountName: prometheus
          securityContext:
            fsGroup: 65534
          containers:
            - name: prometheus
              image: prom/prometheus:v2.54.0
              args:
                - --config.file=/etc/prometheus/prometheus.yml
                - --storage.tsdb.path=/prometheus
                - --web.enable-lifecycle
              ports:
                - containerPort: 9090
              volumeMounts:
                - name: config
                  mountPath: /etc/prometheus
                - name: data
                  mountPath: /prometheus
          volumes:
            - name: config
              configMap:
                name: prometheus-config
            - name: data
              persistentVolumeClaim:
                claimName: prometheus-data
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: prometheus
      namespace: monitoring
    spec:
      selector:
        app: prometheus
      ports:
        - name: web
          port: 9090
          targetPort: 9090
      type: ClusterIP
    
 ###  prometheus-ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: prometheus-ingress
      namespace: monitoring
      annotations:
        cert-manager.io/cluster-issuer: "letsencrypt-prod"
    spec:
      ingressClassName: nginx
      tls:
      - hosts:
        - prometheus.adqget.bar
        secretName: prometheus-tls
      rules:
      - host: prometheus.adqget.bar
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-nodeport   # your Prometheus svc name
                port:
                  number: 9090
    
### node-expoter.yaml  

    # node-exporter.yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: node-exporter
      namespace: monitoring
      labels:
        app: node-exporter
    spec:
      selector:
        matchLabels:
          app: node-exporter
      template:
        metadata:
          labels:
            app: node-exporter
        spec:
          hostNetwork: true
          hostPID: true
          tolerations:
            - operator: "Exists"
          containers:
            - name: node-exporter
              image: quay.io/prometheus/node-exporter:v1.8.0
              args:
                - --path.rootfs=/host
              ports:
                - name: metrics
                  containerPort: 9100
                  hostPort: 9100
                  protocol: TCP
              volumeMounts:
                - name: rootfs
                  mountPath: /host
                  readOnly: true
          volumes:
            - name: rootfs
              hostPath:
                path: /
    
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: node-exporter
      namespace: monitoring
      labels:
        app: node-exporter
    spec:
      selector:
        app: node-exporter
      ports:
        - name: metrics
          port: 9100
          targetPort: 9100
      type: ClusterIP

### kube-state-metrics.yaml

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: kube-state-metrics
      namespace: monitoring
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: kube-state-metrics
    rules:
      # Core resources
      - apiGroups: [""]
        resources:
          - pods
          - services
          - endpoints
          - nodes
          - namespaces
          - replicationcontrollers
          - persistentvolumeclaims
          - persistentvolumes
          - configmaps
          - secrets
          - resourcequotas
          - limitranges
          - events
        verbs: ["list", "watch"]
    
      # Apps API group
      - apiGroups: ["apps"]
        resources:
          - deployments
          - daemonsets
          - replicasets
          - statefulsets
        verbs: ["list", "watch"]
    
      # Networking API group
      - apiGroups: ["networking.k8s.io"]
        resources:
          - networkpolicies
          - ingresses
        verbs: ["list", "watch"]
    
      # Policy API group
      - apiGroups: ["policy"]
        resources:
          - poddisruptionbudgets
        verbs: ["list", "watch"]
    
      # Storage API group
      - apiGroups: ["storage.k8s.io"]
        resources:
          - storageclasses
          - volumeattachments
        verbs: ["list", "watch"]
    
      # Coordination API group
      - apiGroups: ["coordination.k8s.io"]
        resources:
          - leases
        verbs: ["list", "watch"]
    
      # Certificates API group
      - apiGroups: ["certificates.k8s.io"]
        resources:
          - certificatesigningrequests
        verbs: ["list", "watch"]
    
      # Admissionregistration API group
      - apiGroups: ["admissionregistration.k8s.io"]
        resources:
          - mutatingwebhookconfigurations
          - validatingwebhookconfigurations
        verbs: ["list", "watch"]
    
      # Autoscaling API group
      - apiGroups: ["autoscaling"]
        resources:
          - horizontalpodautoscalers
        verbs: ["list", "watch"]
    
      # Batch API group
      - apiGroups: ["batch"]
        resources:
          - jobs
          - cronjobs
        verbs: ["list", "watch"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: kube-state-metrics
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: kube-state-metrics
    subjects:
      - kind: ServiceAccount
        name: kube-state-metrics
        namespace: monitoring
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kube-state-metrics
      namespace: monitoring
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: kube-state-metrics
      template:
        metadata:
          labels:
            app: kube-state-metrics
        spec:
          serviceAccountName: kube-state-metrics
          containers:
          - name: kube-state-metrics
            image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.11.0
            ports:
            - containerPort: 8080
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: kube-state-metrics
      namespace: monitoring
      labels:
        app: kube-state-metrics
    spec:
      type: ClusterIP
      selector:
        app: kube-state-metrics
      ports:
        - port: 8080
          targetPort: 8080
    
### promtail.yaml

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
                resources:
                  - pods
                  - namespaces
                  - nodes
                verbs:
                  - get
                  - list
                  - watch
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
                  log_level: info
            
                positions:
                  filename: /run/promtail/positions.yaml
            
                clients:
                  - url: http://10.10.101.140:3100/loki/api/v1/push
            
                scrape_configs:
                  # --- System logs ---
                  - job_name: varlogs
                    static_configs:
                      - targets: [localhost]
                        labels:
                          job: varlogs
                          __path__: /var/log/**/*.log
            
                  # --- Application logs ---
                  - job_name: app-logs
                    static_configs:
                      - targets: [localhost]
                        labels:
                          job: app-logs
                          namespace: adq-dev
                          __path__: /var/log/adq-dev/*.log
            
                  # --- Kubernetes pod logs (Static discovery for private EKS) ---
                  - job_name: kubernetes-pods-static
                    static_configs:
                      - targets: [localhost]
                        labels:
                          job: kubernetes-pods
                          __path__: /var/log/pods/*/*/*.log
                    pipeline_stages:
                      - regex:
                          expression: '/var/log/pods/(?P<namespace>[^_]+)_(?P<pod>[^_]+)_[^/]+/(?P<container>[^/]+)/.*'
                      - labels:
                          namespace:
                          pod:
                          container:
            ---
            apiVersion: apps/v1
            kind: DaemonSet
            metadata:
              name: promtail
              namespace: monitoring
              labels:
                app: promtail
            spec:
              selector:
                matchLabels:
                  app: promtail
              template:
                metadata:
                  labels:
                    app: promtail
                spec:
                  serviceAccountName: promtail
                  tolerations:
                    - operator: Exists
                  containers:
                    - name: promtail
                      image: grafana/promtail:2.9.0
                      args:
                        - -config.file=/etc/promtail/promtail.yaml
                      volumeMounts:
                        - name: config
                          mountPath: /etc/promtail
                        - name: varlog
                          mountPath: /var/log
                          readOnly: true
                        - name: positions
                          mountPath: /run/promtail
                  volumes:
                    - name: config
                      configMap:
                        name: promtail-config
                    - name: varlog
                      hostPath:
                        path: /var/log
                    - name: positions
                      hostPath:
                        path: /run/promtail
                        type: DirectoryOrCreate



### cAdvisor.yaml

    kind: DaemonSet
    metadata:
      name: cadvisor
      namespace: monitoring
      labels:
        app: cadvisor
    spec:
      selector:
        matchLabels:
          app: cadvisor
      template:
        metadata:
          labels:
            app: cadvisor
        spec:
          containers:
            - name: cadvisor
              image: gcr.io/cadvisor/cadvisor:v0.47.2
              ports:
                - name: http-metrics   # 👈 matches PodMonitor port
                  containerPort: 8080
              volumeMounts:
                - name: rootfs
                  mountPath: /rootfs
                  readOnly: true
                - name: var-run
                  mountPath: /var/run
                  readOnly: false
                - name: sys
                  mountPath: /sys
                  readOnly: true
                - name: docker
                  mountPath: /var/lib/docker
                  readOnly: true
          volumes:
            - name: rootfs
              hostPath:
                path: /
            - name: var-run
              hostPath:
                path: /var/run
            - name: sys
              hostPath:
                path: /sys
            - name: docker
              hostPath:
                path: /var/lib/docker    
 ### ebs-csi-policy.json

     {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "ec2:CreateVolume",
            "ec2:AttachVolume",
            "ec2:DetachVolume",
            "ec2:DeleteVolume",
            "ec2:ModifyVolume",
            "ec2:DescribeVolumes",
            "ec2:DescribeVolumeAttribute",
            "ec2:DescribeInstances",
            "ec2:DescribeAvailabilityZones",
            "ec2:CreateTags",
            "ec2:DescribeSnapshots",
            "ec2:CreateSnapshot",
            "ec2:DeleteSnapshot",
            "ec2:DescribeTags"
          ],
          "Resource": "*"
        }
      ]
    }

# loki - configuration:


### cat /ete/loki/loki-config.yaml
                
     server:
      http_listen_port: 3100
      http_listen_address: 0.0.0.0
      log_level: info
    
    auth_enabled: false   # disable multi-tenant auth
    
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
      compaction_interval: 5m
    
    limits_config:
      ingestion_rate_mb: 8
      ingestion_burst_size_mb: 16
      max_entries_limit_per_query: 5000
      max_streams_per_user: 10000
      reject_old_samples: true
      retention_period: 168h   # 7 days retention
    
    chunk_store_config:
      max_look_back_period: 0s
    
    table_manager:
      retention_deletes_enabled: true
      retention_period: 168h
    
         

    
### /etc/systemd/system/ loki.service
     

    [Unit]
    Description=Loki Log Aggregator
    After=network.target
    
    [Service]
    User=loki
    Group=loki
    Type=simple
    
    ExecStart=/usr/local/bin/loki \
      -config.file=/etc/loki/loki-config.yaml \
      -server.http-listen-address=0.0.0.0:3100 \
      -server.grpc-listen-address=0.0.0.0:9095
    
    Restart=always
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target


# Grafana:

### /etc/systemd/system/grafana.service

    [Unit]
    Description=Grafana
    After=network.target
    
    [Service]
    Type=simple
    ExecStart=/usr/share/grafana/bin/grafana-server web
    Restart=on-failure
    User=root
    WorkingDirectory=/usr/share/grafana
    
    [Install]
    WantedBy=multi-user.target

# Promethus-stack.yaml

    ---
    # ServiceAccount for Prometheus
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: prometheus
      namespace: monitoring
    ---
    # RBAC for Prometheus
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: prometheus
    rules:
      - apiGroups: [""]
        resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods", "namespaces", "events"]
        verbs: ["get", "list", "watch", "create", "patch"]
      - apiGroups: ["apps", "extensions"]
        resources: ["deployments", "replicasets", "daemonsets", "statefulsets"]
        verbs: ["get", "list", "watch"]
      - apiGroups: ["networking.k8s.io"]
        resources: ["ingresses"]
        verbs: ["get", "list", "watch"]
      - apiGroups: ["coordination.k8s.io"]
        resources: ["leases"]
        verbs: ["get", "list", "watch", "create", "update", "patch"]
      - apiGroups: ["monitoring.coreos.com"]
        resources: ["prometheuses","alertmanagers","servicemonitors","prometheusrules","thanosrulers","podmonitors","scrapeconfigs"]
        verbs: ["get","list","watch","create","update","patch"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: prometheus
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: prometheus
    subjects:
      - kind: ServiceAccount
        name: prometheus
        namespace: monitoring
    ---
    # Prometheus CR (Operator will create StatefulSet)
    apiVersion: monitoring.coreos.com/v1
    kind: Prometheus
    metadata:
      name: prometheus
      namespace: monitoring
    spec:
      replicas: 1
      serviceAccountName: prometheus
      serviceMonitorSelector: {}
      podMonitorSelector: {}
      resources:
        requests:
          memory: 400Mi
          cpu: 200m
      retention: 10d
      enableAdminAPI: true
      securityContext:
        runAsUser: 65534
        runAsGroup: 65534
        fsGroup: 65534
      storage:
        volumeClaimTemplate:
          metadata:
            name: prometheus-data
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 10Gi
            storageClassName: gp3
    ---
    # Prometheus Service
    apiVersion: v1
    kind: Service
    metadata:
      name: prometheus-service
      namespace: monitoring
      labels:
        app.kubernetes.io/name: prometheus
        app.kubernetes.io/instance: prometheus
    spec:
      type: ClusterIP
      ports:
      - name: web
        port: 9090
        targetPort: web
      selector:
        prometheus: prometheus
    ---
    # Ingress for Prometheus
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: prometheus-ingress
      namespace: monitoring
      annotations:
        cert-manager.io/cluster-issuer: "letsencrypt-prod"
    spec:
      ingressClassName: nginx
      tls:
        - hosts:
            - prometheus.adqget.bar
          secretName: prometheus-tls
      rules:
        - host: prometheus.adqget.bar
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: prometheus-service
                    port:
                      number: 9090
    ---
    # ServiceMonitor for Node Exporter
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: node-exporter
      namespace: monitoring
    spec:
      selector:
        matchLabels:
          app: node-exporter
      namespaceSelector:
        matchNames:
          - monitoring
      endpoints:
      - port: metrics
        interval: 15s
    ---
    # ServiceMonitor for Kube State Metrics
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: kube-state-metrics
      namespace: monitoring
    spec:
      selector:
        matchLabels:
          app: kube-state-metrics
      namespaceSelector:
        matchNames:
          - monitoring
      endpoints:
      - port: http-metrics
        interval: 15s
    ---
    # PodMonitor for cAdvisor
    apiVersion: monitoring.coreos.com/v1
    kind: PodMonitor
    metadata:
      name: cadvisor
      namespace: monitoring
    spec:
      selector:
        matchLabels:
          app: cadvisor
      namespaceSelector:
        matchNames:
          - monitoring
      podMetricsEndpoints:
      - port: http-metrics
        interval: 15s
    ---
    # Prometheus Rules for Cluster Usage
    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: cluster-rules
      namespace: monitoring
    spec:
      groups:
      - name: cluster.rules
        rules:
        - record: instance:node_memory_utilisation:ratio
          expr: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
        - record: instance:node_cpu_utilisation:rate5m
          expr: 1 - avg without (cpu) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
        - record: instance:node_disk_utilisation:rate5m
          expr: rate(node_disk_io_time_seconds_total[5m])

# loki dashbaord - json -> import dashboard

    {
      "title": "Unified EKS Logs (System + K8s + App)",
      "uid": "eks-logs-unified-v2",
      "timezone": "browser",
      "schemaVersion": 38,
      "version": 1,
      "refresh": "10s",
      "tags": ["loki","logs","kubernetes","promtail"],
      "templating": {
        "list": [
          {
            "name": "job",
            "type": "query",
            "datasource": "loki",
            "refresh": 1,
            "query": "label_values(job)",
            "includeAll": true,
            "multi": true,
            "allValue": ".+",
            "current": { "text": "All", "value": ".+" }
          },
          {
            "name": "namespace",
            "type": "query",
            "datasource": "loki",
            "refresh": 1,
            "query": "label_values({job=~\"$job\"}, namespace)",
            "includeAll": true,
            "multi": true,
            "allValue": ".+",
            "current": { "text": "All", "value": ".+" }
          },
          {
            "name": "pod",
            "type": "query",
            "datasource": "loki",
            "refresh": 1,
            "query": "label_values({job=~\"$job\", namespace=~\"$namespace\"}, pod)",
            "includeAll": true,
            "multi": true,
            "allValue": ".+",
            "current": { "text": "All", "value": ".+" }
          },
          {
            "name": "container",
            "type": "query",
            "datasource": "loki",
            "refresh": 1,
            "query": "label_values({job=~\"$job\", namespace=~\"$namespace\", pod=~\"$pod\"}, container)",
            "includeAll": true,
            "multi": true,
            "allValue": ".+",
            "current": { "text": "All", "value": ".+" }
          }
        ]
      },
      "panels": [
        {
          "type": "timeseries",
          "title": "Log volume by job",
          "datasource": "loki",
          "targets": [
            { "expr": "sum by (job) (rate({job=~\"$job\"}[1m]))", "legendFormat": "{{job}}" }
          ],
          "fieldConfig": { "defaults": { "unit": "ops" } },
          "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 }
        },
        {
          "type": "timeseries",
          "title": "Log volume by namespace",
          "datasource": "loki",
          "targets": [
            { "expr": "sum by (namespace) (rate({job=~\"$job\", namespace=~\"$namespace\"}[1m]))", "legendFormat": "{{namespace}}" }
          ],
          "fieldConfig": { "defaults": { "unit": "ops" } },
          "gridPos": { "x": 12, "y": 0, "w": 12, "h": 8 }
        },
        {
          "type": "table",
          "title": "Top pods by log lines",
          "datasource": "loki",
          "targets": [
            { "expr": "topk(10, sum by (pod) (rate({job=~\"$job\", namespace=~\"$namespace\"}[5m])))", "legendFormat": "{{pod}}" }
          ],
          "gridPos": { "x": 0, "y": 8, "w": 12, "h": 7 }
        },
        {
          "type": "table",
          "title": "Top containers by log lines",
          "datasource": "loki",
          "targets": [
            { "expr": "topk(10, sum by (container) (rate({job=~\"$job\", namespace=~\"$namespace\", pod=~\"$pod\"}[5m])))", "legendFormat": "{{container}}" }
          ],
          "gridPos": { "x": 12, "y": 8, "w": 12, "h": 7 }
        },
        {
          "type": "logs",
          "title": "Live logs",
          "datasource": "loki",
          "options": { "showLabels": true, "showTime": true, "wrapLogMessage": true },
          "targets": [
            { "expr": "{job=~\"$job\", namespace=~\"$namespace\", pod=~\"$pod\", container=~\"$container\"}" }
          ],
          "gridPos": { "x": 0, "y": 15, "w": 24, "h": 12 }
        }
      ]
    }

 
 # Name Spaces   
    [root@ip-10-10-101-140 ~]# kubectl get ns
    NAME              STATUS   AGE
    adq-dev           Active   19d
    cert-manager      Active   17d
    default           Active   21d
    external-dns      Active   21d
    ingress-nginx     Active   19d
    kube-node-lease   Active   21d
    kube-public       Active   21d
    kube-system       Active   21d
    kyverno           Active   15d
    monitoring        Active   9d
