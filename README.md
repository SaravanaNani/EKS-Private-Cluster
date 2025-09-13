# EKS Cluster Setup with Bastion, Ingress, and Sample App Deployment

This document provides step-by-step instructions to set up an AWS EKS cluster with private worker nodes, a Bastion host for secure access, NGINX Ingress, a sample application, and GoDaddy DNS integration. All naming conventions are preserved as provided.

---

## Step 0 — Preparation

* Project: `adq`
* Environment: `dev` | `prod`
* AWS Account: Ensure admin IAM user
* Enable billing alerts
* Create key pair: `adq-keypair-<region>` for EC2 access

---

## Step 1 — Create VPC & Subnets

### 1.1 Create VPC

* Name: `adq-dev-vpc`
* CIDR: `10.10.0.0/16`

### 1.2 Subnets

| Type    | CIDR           | AZ | Name              |
| ------- | -------------- | -- | ----------------- |
| Private | 10.10.1.0/24   | a  | adq-dev-private-a |
| Private | 10.10.2.0/24   | b  | adq-dev-private-b |
| Public  | 10.10.101.0/24 | a  | adq-dev-public-a  |
| Public  | 10.10.102.0/24 | b  | adq-dev-public-b  |

* Attach Internet Gateway: `adq-dev-igw`
* Create NAT Gateway: `adq-dev-nat` (in public-a)

### 1.3 Route Tables

| Route Table         | Default Route           | Associated Subnets                   |
| ------------------- | ----------------------- | ------------------------------------ |
| adq-dev-rtb-public  | 0.0.0.0/0 → adq-dev-igw | adq-dev-public-a, adq-dev-public-b   |
| adq-dev-rtb-private | 0.0.0.0/0 → adq-dev-nat | adq-dev-private-a, adq-dev-private-b |

**Note:** Public subnets can access the internet directly. Private subnets route through NAT for outbound access only. Use a Bastion host for secure access to private nodes.

### 1.4 Security Groups

**1.4.1 Bastion SG: `adq-dev-sg-bastion`**

* Ingress: TCP 22 → your office/public IP

**1.4.2 EKS Control Plane SG: `adq-dev-sg-eks-controlplane`**

* Ingress: 443, 10250 from EKS nodes

**1.4.3 EKS Nodes SG: `adq-dev-sg-eks-nodes`**

* Ingress: 443 → `adq-dev-sg-eks-controlplane`
  10250 → `adq-dev-sg-eks-nodes`
  30000–32767 → VPC CIDR 10.10.0.0/16
  22 → `adq-dev-sg-bastion`

**1.4.4 DB SG: `adq-dev-sg-db`**

* Ingress: 3306 → `adq-dev-sg-eks-nodes`
  3306 → `adq-dev-sg-bastion` (optional)

---

## Step 2 — IAM Roles & EKS Cluster Setup

### 2.1 Create Worker Node Role: `adq-dev-iam-eks-nodes`

* Managed policies:

  * AmazonEKSWorkerNodePolicy
  * AmazonEC2ContainerRegistryReadOnly
  * AmazonEKS\_CNI\_Policy

### 2.2 Create Cluster Role: `adq-dev-iam-eks-cluster`

* Managed policies:

  * AmazonEKSClusterPolicy
  * AmazonEKSServicePolicy

### 2.3 Bastion Role: `adq-dev-eks-admin-role-bastion`

* Managed policies:

  * AmazonEKSClusterPolicy
  * AmazonEKSWorkerNodePolicy
  * AmazonEKS\_CNI\_Policy
  * AmazonEC2ContainerRegistryReadOnly
  * (Optional) AdministratorAccess

### 2.4 EKS Access Entry for Bastion

```bash
aws eks create-access-entry \
  --cluster-name adq-dev-eks \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/adq-dev-eks-admin-role-bastion \
  --type STANDARD

aws eks associate-access-policy \
  --cluster-name adq-dev-eks \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/adq-dev-eks-admin-role-bastion \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

### 2.5 Create EKS Cluster (Control Plane)

* Name: `adq-dev-eks`
* Kubernetes version: latest stable (1.29/1.30)
* IAM Role: `adq-dev-iam-eks-cluster`
* VPC: `adq-dev-vpc`
* Subnets: `adq-dev-private-a`, `adq-dev-private-b`
* SG: `adq-dev-sg-eks-controlplane`
* Endpoint access: Private ON, Public OFF or restricted
* Logging: Enable all

### 2.6 Create Node Group (Worker Nodes)

* Name: `adq-dev-ng`
* IAM Role: `adq-dev-iam-eks-nodes`
* Subnets: private
* Instance type: `t3.medium` (dev)
* Disk: 20 GiB
* Scaling: Min=2, Desired=2, Max=4
* SSH: Optional via Bastion only
* SG: `adq-dev-sg-eks-nodes`

---

## Step 3 — Bastion Setup

* EC2: `adq-dev-bastion`
* AMI: Amazon Linux 2
* Type: t3.micro
* Subnet: `adq-dev-public-a`
* SG: `adq-dev-sg-bastion`
* Install tools: aws-cli, kubectl, helm, git, curl
* Update kubeconfig:

```bash
aws eks update-kubeconfig --region ap-south-1 --name adq-dev-eks-cluster
kubectl get nodes
```

* Verify IAM role and metadata service with IMDSv2:

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/adq-dev-eks-admin-role-bastion
aws sts get-caller-identity
```

---

## Step 4 — Helm & NGINX Ingress Setup

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
kubectl create namespace ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"=internet-facing
kubectl get service --namespace ingress-nginx ingress-nginx-controller --output wide --watch
```

---

## Step 5 — Deploy Sample App (hello-k8s)

```yaml
# hello-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-k8s
  namespace: adq-dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-k8s
  template:
    metadata:
      labels:
        app: hello-k8s
    spec:
      containers:
      - name: hello-k8s
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-k8s
  namespace: adq-dev
spec:
  selector:
    app: hello-k8s
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f hello-deployment.yaml
kubectl get pods -n adq-dev
kubectl get svc -n adq-dev
kubectl port-forward svc/hello-k8s -n adq-dev 8080:80
curl http://localhost:8080
```

---

## Step 6 — Ingress Resource

```yaml
# hello-k8s-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-k8s-ingress
  namespace: adq-dev
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: app.adqget.bar
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-k8s
            port:
              number: 80
```

```bash
kubectl apply -f hello-k8s-ingress.yaml
kubectl get ingress -n adq-dev
```

* GoDaddy: Point CNAME `app` → ingress NLB hostname
* Verify DNS propagation:

```bash
nslookup app.adqget.bar
dig +short app.adqget.bar
```

---

## Notes

* Security groups: Ingress rules are sufficient; egress uses defaults.
* EKS endpoint access: Private ON, Public OFF (or restricted)
* Route tables: Ensure private subnets have NAT for outbound access.
* Network diagram: Public Subnets → IGW, Private Subnets → NAT → IGW, Bastion → Nodes/DB
* All naming conventions from `adq-dev-*` are followed.

---

This README consolidates the entire process for real-world EKS cluster setup, Bastion access, ingress setup, sample app deployment, and DNS integration.
