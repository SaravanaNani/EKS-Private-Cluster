# 🚀 My App Helm Chart

This Helm chart deploys the **DevOps Task App** (`saravana2002/demo-node-logger`) to a Kubernetes cluster with support for TLS Ingress, environment variables, and centralized log storage.

---

## 📁 Folder Structure

```
my-app-helm/
│
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── _helpers.tpl
```

---

## 📦 Chart.yaml

```yaml
apiVersion: v2
name: my-app-helm
description: A Helm chart for deploying the DevOps Task App
type: application
version: 0.1.0
appVersion: "1.0.0"
```

---

## ⚙️ values.yaml

```yaml
replicaCount: 2

image:
  repository: saravana2002/demo-node-logger
  tag: latest
  pullPolicy: Always

containerPort: 3000
service:
  type: ClusterIP
  port: 80

namespace: adq-dev
appName: devops-task-app

env:
  - name: NODE_ENV
    value: production

logPath: /var/log/adq-dev

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: app.adqget.bar
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: devops-task-tls
      hosts:
        - app.adqget.bar
```

---

## 🧩 templates/_helpers.tpl

```yaml
{{/*
Generate a fullname for resources
*/}}
{{- define "my-app-helm.fullname" -}}
{{- printf "%s-%s" .Release.Name .Values.appName | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

---

## 🚀 templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app-helm.fullname" . }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.appName }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      containers:
        - name: {{ .Values.appName }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.containerPort }}
          env:
            {{- range .Values.env }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
          command: ["/bin/sh", "-c"]
          args:
            - |
              mkdir -p /usr/src/app/logs &&               node app.js 2>&1 | tee -a /usr/src/app/logs/app.log
          volumeMounts:
            - name: app-logs
              mountPath: /usr/src/app/logs
      volumes:
        - name: app-logs
          hostPath:
            path: {{ .Values.logPath }}
            type: DirectoryOrCreate
```

---

## 🌐 templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app-helm.fullname" . }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.appName }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Values.appName }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}
```

---

## 🔒 templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app-helm.fullname" . }}-ingress
  namespace: {{ .Values.namespace }}
  annotations:
    {{- range $key, $value := .Values.ingress.annotations }}
    {{ $key }}: {{ $value | quote }}
    {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "my-app-helm.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

---

## 🧠 Usage Commands

```bash
# 1️⃣ Create the chart directory
mkdir -p my-app-helm/templates

# 2️⃣ Save the above files in their respective locations.

# 3️⃣ Deploy to your cluster
helm install devops-task ./my-app-helm --namespace adq-dev --create-namespace

# 4️⃣ To upgrade
helm upgrade devops-task ./my-app-helm

# 5️⃣ To uninstall
helm uninstall devops-task -n adq-dev
```

---

## 💡 Optional Enhancement

You can replace `hostPath` with a PersistentVolumeClaim (PVC) for storing logs in cloud environments like EKS, GKE, or AKS.

---
# 🚀 Helm Chart Packaging & Publishing Guide (GitHub Pages)

This document explains how to **package**, **host**, and **publish** a Helm chart publicly on **GitHub Pages**, so anyone can install it using `helm repo add`.

---

## 🧭 Overview

You’ll learn how to:

1. Package your Helm chart (`.tgz` file)
2. Generate the Helm repository index
3. Push it to GitHub
4. Serve it via GitHub Pages  
5. Test and verify your public Helm repository

---

## 📁 Folder Structure

Example structure inside your GitHub repo (`EKS-Private-Cluster`):

```
EKS-Private-Cluster/
├── helm-charts/
│   ├── my-app-helm-0.1.0.tgz
│   └── index.yaml
├── Monitoring.md
├── README.md
└── other-docs...
```

---

## ⚙️ Step 1 — Package Your Helm Chart

From your local folder containing `my-app-helm/`:

```bash
helm package my-app-helm
```

This creates a compressed chart file:

```
my-app-helm-0.1.0.tgz
```

Now move it into the `helm-charts/` folder inside your GitHub repository:

```bash
mv my-app-helm-0.1.0.tgz EKS-Private-Cluster/helm-charts/
```

---

## 🧾 Step 2 — Create the Helm Repository Index

Change into the `helm-charts` folder:

```bash
cd EKS-Private-Cluster/helm-charts
```

Then run:

```bash
helm repo index . --url https://saravananani.github.io/EKS-Private-Cluster/helm-charts
```

✅ This generates an `index.yaml` file, which tells Helm what charts are available and their download URLs.

You should now have:
```
helm-charts/
├── index.yaml
└── my-app-helm-0.1.0.tgz
```

---

## 🔐 Step 3 — Push Files to GitHub

Go back to your repo root and push your changes:

```bash
cd ..
git add helm-charts
git commit -m "Add Helm chart and repository index"
git push origin main
```

> ⚠️ If prompted for GitHub credentials:
> - Use your **GitHub username**
> - Use a **Personal Access Token (PAT)** instead of your password
> (Password authentication is disabled for Git operations.)

---

## 🔑 Step 4 — Configure GitHub Pages

1. Go to your GitHub repository  
   👉 [https://github.com/SaravanaNani/EKS-Private-Cluster](https://github.com/SaravanaNani/EKS-Private-Cluster)

2. Navigate to:  
   **Settings → Pages**

3. Under **Source**, choose:
   - **Branch:** `main`
   - **Folder:** `/helm-charts`

4. Click **Save**

🕒 Wait about 1–2 minutes for GitHub Pages to publish your files.

---

## 🌍 Step 5 — Verify Your Hosted Chart

Check in your browser:

```
https://saravananani.github.io/EKS-Private-Cluster/helm-charts/index.yaml
```

✅ You should see YAML content like:

```yaml
apiVersion: v1
entries:
  my-app-helm:
  - apiVersion: v2
    name: my-app-helm
    version: 0.1.0
    urls:
    - https://saravananani.github.io/EKS-Private-Cluster/helm-charts/my-app-helm-0.1.0.tgz
```

That confirms your Helm repository is live and public 🎉

---

## 🧪 Step 6 — Test With Helm CLI

Now test adding your repository and installing the chart.

```bash
# Add your new Helm repo
helm repo add myrepo https://saravananani.github.io/EKS-Private-Cluster/helm-charts/

# Update local repo cache
helm repo update

# Verify your chart appears
helm search repo myrepo

# Install the chart
helm install devops-task myrepo/my-app-helm -n adq-dev --create-namespace
```

✅ You should see:
```
myrepo/my-app-helm   0.1.0   A Helm chart for deploying the DevOps Task App
```

---

## 🧱 Step 7 — Update or Publish a New Version

When you make changes to your chart:

1. Update `version:` in `my-app-helm/Chart.yaml`
2. Re-package:
   ```bash
   helm package my-app-helm
   mv my-app-helm-0.1.1.tgz helm-charts/
   cd helm-charts
   helm repo index . --url https://saravananani.github.io/EKS-Private-Cluster/helm-charts --merge index.yaml
   ```
3. Commit and push:
   ```bash
   git add .
   git commit -m "Release version 0.1.1"
   git push origin main
   ```

Your GitHub Pages site automatically updates the index with the new chart.

---

## 🧠 Optional Tips

- To keep credentials saved:
  ```bash
  git config --global credential.helper store
  ```
- To view all added Helm repos:
  ```bash
  helm repo list
  ```
- To remove old versions from Helm cache:
  ```bash
  helm repo remove myrepo
  ```

---

## ✅ Summary

| Step | Command / Action | Description |
|------|-------------------|--------------|
| 1 | `helm package my-app-helm` | Creates chart archive |
| 2 | `helm repo index . --url <URL>` | Generates repository index |
| 3 | `git push origin main` | Push chart and index to GitHub |
| 4 | Enable GitHub Pages | Serve `/helm-charts/` folder |
| 5 | `helm repo add myrepo <URL>` | Add repo to Helm client |
| 6 | `helm install ...` | Deploy your app via Helm |

---

## 🎯 Final Repository URL

Your public Helm repository is now available at:

👉 **[https://saravananani.github.io/EKS-Private-Cluster/helm-charts/](https://saravananani.github.io/EKS-Private-Cluster/helm-charts/)**

You can share this URL with anyone — they can add your Helm repo and install your chart directly!
