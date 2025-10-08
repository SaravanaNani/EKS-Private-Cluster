# TLS Certificate Setup and Application Deployment on EKS – Complete Guide

This guide documents, end-to-end, how we deployed a sample application to **Amazon EKS**, exposed it via **NGINX Ingress**, and secured it with **TLS certificates from Let’s Encrypt** using **cert-manager**. It also covers the **DNS (GoDaddy)** setup and explains *why* each component is used.

> **Domain used in examples:** `app.adqget.bar`  
> **Namespace:** `adq-dev`  
> **Ingress class:** `nginx`  
> **ClusterIssuer:** `letsencrypt-prod`

---

## 1) Overview

### What is TLS and why we use it
- **TLS (Transport Layer Security)** encrypts traffic between clients and your application so data (cookies, tokens, PII) isn’t exposed to attackers.
- A **valid certificate** binds your domain name to your server, enabling browsers to trust your site (padlock icon).

### Why cert-manager + Let’s Encrypt
- **cert-manager** automates certificate **issuance**, **renewal**, and **storage** as Kubernetes **Secrets**.
- **Let’s Encrypt** is a free, trusted CA that supports automation via **ACME** protocol.
- Result: no manual certificate creation/renewal; Kubernetes native workflow.

### High-level architecture
```
Client (Browser)
   │ HTTPS:443
   ▼
AWS NLB (from ingress-nginx Service type=LoadBalancer)
   ▼
NGINX Ingress Controller (in cluster, class=nginx)
   ▼
TLS termination (uses Secret created by cert-manager)
   ▼
Service (ClusterIP) → Pod(s) (your app)
```

- **TLS termination** happens at the **Ingress** (NGINX). The certificate is provisioned by cert-manager and stored as a Secret referenced by the Ingress.
- **Let’s Encrypt HTTP-01 Challenge**: cert-manager temporarily creates an HTTP path `/.well-known/acme-challenge/*` on your Ingress so Let’s Encrypt can verify domain ownership.

---

## 2) Pre-requisites

- An **EKS** cluster up and running (worker nodes Ready).
- **kubectl** and **helm** configured to talk to your cluster.
- **NGINX Ingress Controller** installed (creates a Service of type **LoadBalancer**, backed by an **AWS NLB**). Example install (already done previously):
  ```bash
  kubectl create namespace ingress-nginx
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm repo update
  helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx     --set controller.service.type=LoadBalancer     --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb     --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"=internet-facing
  ```
- A **public domain** (e.g., managed in GoDaddy) that you can point to your Ingress **NLB hostname**.
- Internet-facing Ingress (ACME HTTP-01 requires Let’s Encrypt to reach `http://<your-domain>/.well-known/acme-challenge/...`).

---

## 3) cert-manager Setup

> **Why**: Automates requesting, validating, issuing, and renewing certificates in K8s.

### 3.1 Install CRDs + Helm chart
```bash
# Install CRDs
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml

# Install cert-manager via Helm
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager   --namespace cert-manager   --version v1.14.4
```

### 3.2 Create a ClusterIssuer (Let’s Encrypt Production)
> **Why**: The ClusterIssuer defines the ACME server (Let’s Encrypt), contact email, and the solver (HTTP-01 via nginx).

Save as **`clusterissuer-letsencrypt-prod.yaml`**:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Let’s Encrypt production endpoint (real certs, with rate limits)
    server: https://acme-v02.api.letsencrypt.org/directory
    email: saravana09052002aws@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

Apply:
```bash
kubectl apply -f clusterissuer-letsencrypt-prod.yaml
kubectl get clusterissuer letsencrypt-prod -o yaml
```

**How ACME HTTP-01 works (short):**
1. You create an Ingress with the annotation `cert-manager.io/cluster-issuer: letsencrypt-prod` and a `tls:` section.
2. cert-manager creates a temporary **challenge** Ingress to serve a **token** under `/.well-known/acme-challenge/...`.
3. Let’s Encrypt calls `http://app.adqget.bar/.well-known/acme-challenge/<token>` to verify domain control.
4. If reachable and correct, Let’s Encrypt issues the certificate. cert-manager saves it into a Kubernetes **Secret**.
5. cert-manager automatically **renews** the cert before it expires (default: 30 days prior).

---

## 4) DNS Configuration (GoDaddy)

> **Why**: Your domain must resolve to the Ingress load balancer so Let’s Encrypt and clients can reach the app.

### 4.1 Get the Ingress NLB hostname
```bash
# Get NLB hostname created by ingress-nginx
kubectl get svc -n ingress-nginx ingress-nginx-controller   -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
# Example output:
# a90a822f55c1c431187ff91e68b51d34-e6e10ddc1a78aa45.elb.us-east-1.amazonaws.com
```

### 4.2 Create a DNS record in GoDaddy
- **Type**: CNAME  
- **Name**: `app` (so it becomes `app.adqget.bar`)  
- **Value/Target**: `<the NLB hostname from above>`

### 4.3 Verify DNS
```bash
nslookup app.adqget.bar
dig +short app.adqget.bar
# Should resolve to the NLB hostname and/or IPs returned by AWS
```

> **Note:** DNS propagation may take a few minutes. cert-manager’s ACME flow will not succeed until the domain resolves correctly to your Ingress.

---

## 5) Application Build & Push (Example)

> **Why**: We need a containerized application image to deploy to Kubernetes.

### 5.1 Example Dockerfile (Node.js)
Save as **`Dockerfile`**:

```dockerfile# Use lightweight Node.js image

FROM node:18-alpine

# Working directory inside container
WORKDIR /usr/src/app

# Copy app files
COPY . .

# Install minimal dependencies
RUN npm install express

# Expose port
EXPOSE 3000

# Start app
CMD ["node", "app.js"]r.js"]
```

save as **`app.js`**"
```
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  console.log(`[${new Date().toISOString()}] GET / request received`);
  res.send('🚀 Sample Node App for Promtail Logging Demo!');
});

setInterval(() => {
  console.log(`[${new Date().toISOString()}] App heartbeat log`);
}, 5000);

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```
### 5.2 Build, tag, and push (Docker Hub example)

```bash
# login first
docker login

# build
docker build -t <dockerhub_user>/demo-node-logger:latest .

# push
docker push <dockerhub_user>/demo-node-logger:latest
```

> In our earlier test we used: `saravana2002/devops-task:latest` (listens on port **3000** in the container).

---

## 6) Kubernetes Deployment (Namespace: `adq-dev`)

> **Why**: Deploy the app as Pods and expose it with a Service (ClusterIP) that the Ingress will route to.

Save as **`app-deployment.yaml`**:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: adq-dev
---
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
        image: saravana2002/devops-task:latest   # or <dockerhub_user>/demo-node-logger:latest
        ports:
        - containerPort: 3000   # app listens in container
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
    - port: 80          # service port
      targetPort: 3000  # forwards to containerPort
  type: ClusterIP       # behind ingress
```

Apply and verify:
```bash
kubectl apply -f app-deployment.yaml
kubectl get pods -n adq-dev -o wide
kubectl get svc  -n adq-dev
kubectl logs -n adq-dev deploy/devops-task-app --tail=50
```

---

## 7) Ingress with TLS (cert-manager + Let’s Encrypt)

> **Why**: Terminate HTTPS at the ingress with a cert that cert-manager provisions and renews automatically.

Save as **`app-ingress-tls.yaml`**:
```yaml
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
    secretName: tls-app-adqget-bar   # cert-manager will create/manage this Secret
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
```

Apply and check:
```bash
kubectl apply -f app-ingress-tls.yaml

# Watch certificate request/issuance
kubectl get certificate,order,challenge -A

# Inspect the created Secret when ready
kubectl get secret tls-app-adqget-bar -n adq-dev -o yaml

# Check ingress status + address
kubectl get ingress -n adq-dev
kubectl describe ingress devops-task-ingress -n adq-dev
```

**How the certificate is created & renewed**
- cert-manager notices the Ingress annotation + `tls:` block.
- Creates a **Certificate** resource → triggers an **Order** and **Challenge**.
- Serves the ACME HTTP-01 token using your NGINX ingress.
- After validation, the real cert is stored in **Secret** `tls-app-adqget-bar` and attached to the Ingress.
- cert-manager automatically renews the cert and updates the Secret (Ingress picks it up with no downtime).

---

## 8) Validation

### 8.1 Browser
- Visit **`https://app.adqget.bar`** → should show your app with a valid padlock.

### 8.2 CLI
```bash
# Resolve DNS
nslookup app.adqget.bar
dig +short app.adqget.bar

# Check Kubernetes resources
kubectl get pods -n adq-dev
kubectl get svc  -n adq-dev
kubectl get ingress -n adq-dev

# Inspect cert from outside
openssl s_client -connect app.adqget.bar:443 -servername app.adqget.bar </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

---

## 9) Troubleshooting

### Cert stuck in Pending
- **DNS not ready**: Ensure `app.adqget.bar` resolves to the **NLB hostname** of ingress-nginx.
- **Ingress class mismatch**: Your ClusterIssuer solver uses `ingress.class: nginx`. Ensure your Ingress uses `ingressClassName: nginx`.
- **Multiple ingress controllers**: Make sure only NGINX handles this host.

### 404 on `/.well-known/acme-challenge/*`
- Ingress must be **internet-facing** during validation (HTTP-01).
- NLB SG must allow inbound **80/443** (created automatically for internet-facing NLB).
- No conflicting Ingress for the same host/path.

### NLB shows unhealthy / timeouts
- App Service must be **ClusterIP** and selectors must match your Deployment labels.
- Pods must be **Ready** and listening on the correct `containerPort`.
- If curl from bastion to the Service cluster IP works but external fails, check NLB target health & node SG rules.

### DNS propagation
- It may take a few minutes. Verify with:
  ```bash
  dig +short app.adqget.bar
  ```

### Renewal checks
```bash
# See cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager --tail=200

# Describe cert
kubectl describe certificate -n adq-dev tls-app-adqget-bar

# Force re-issue (delete certificate Secret — cert-manager will recreate)
kubectl delete secret -n adq-dev tls-app-adqget-bar
```

---

## 10) Future Enhancements

- **Staging vs Production issuers**: Use Let’s Encrypt **staging** first to avoid prod rate limits.
  ```yaml
  # Staging issuer
  apiVersion: cert-manager.io/v1
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-staging
  spec:
    acme:
      server: https://acme-staging-v02.api.letsencrypt.org/directory
      email: you@example.com
      privateKeySecretRef:
        name: letsencrypt-staging
      solvers:
      - http01:
          ingress:
            class: nginx
  ```
- **Wildcard certificates**: Use **DNS-01** challenges with your DNS provider (requires API credentials) to issue `*.adqget.bar` for multiple subdomains.
- **Central cert management**: Reuse the same ClusterIssuer across namespaces/apps; cert-manager will manage each Ingress Secret independently.
- **Auto-rotation policies**: cert-manager handles renewals automatically; monitor expiry via Prometheus/Grafana alerts if desired.

---

## Command Recap (Quick Start)

```bash
# 1) Install cert-manager + CRDs
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io && helm repo update
helm install cert-manager jetstack/cert-manager -n cert-manager --version v1.14.4

# 2) Create ClusterIssuer (Let’s Encrypt Prod)
kubectl apply -f clusterissuer-letsencrypt-prod.yaml

# 3) Create/verify DNS CNAME: app.adqget.bar → <ingress NLB hostname>
kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
# then set CNAME in GoDaddy and verify:
nslookup app.adqget.bar

# 4) Deploy app + Service
kubectl apply -f app-deployment.yaml

# 5) Create Ingress with TLS annotation
kubectl apply -f app-ingress-tls.yaml

# 6) Validate
kubectl get certificate,order,challenge -A
kubectl get ingress -n adq-dev
openssl s_client -connect app.adqget.bar:443 -servername app.adqget.bar </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

---

### Appendix: Example Manifests Used in This Guide
- `clusterissuer-letsencrypt-prod.yaml`
- `app-deployment.yaml`
- `app-ingress-tls.yaml`

All commands and YAMLs above are **copy-paste runnable** and aligned with the names we used (`app.adqget.bar`, `adq-dev`, `letsencrypt-prod`, ingress `nginx`).

Good luck and happy shipping! 🚀
