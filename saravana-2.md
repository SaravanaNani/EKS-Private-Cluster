# **Saravana L**
**DevOps & Cloud Engineer | CI/CD Automation | Infrastructure as Code (IaC)**  
📞 +91 9390441805 | ✉ saravana09052002aws@gmail.com  
🔗 [LinkedIn: www.linkedin.com/in/saravanal](https://www.linkedin.com/in/saravanal) | 📍 Chittoor, Andhra Pradesh – 517001

---

## 💼 Professional Summary
DevOps & Cloud Engineer with 2+ years of hands-on experience in cloud automation, infrastructure provisioning, and CI/CD pipeline engineering. Skilled at designing and operating Kubernetes (EKS) platforms, building observability pipelines (Prometheus, Grafana, Loki, Promtail), and automating infrastructure with Terraform and Ansible. Strong scripting background in Python used for validation, automation, and operational tooling.

---

## 🛠 Technical Skills
- **Cloud Platforms:** AWS, GCP  
- **IaC & Automation:** Terraform, Ansible, eksctl  
- **Containerization & Orchestration:** Docker, Kubernetes (EKS), Helm  
- **CI/CD & Dev Tools:** Jenkins, GitHub Actions, SonarQube, Nexus, DockerHub, ECR  
- **Monitoring & Logging:** Prometheus (Operator), Grafana, Loki, Promtail, cAdvisor, Node Exporter, kube-state-metrics  
- **Storage & Infra:** AWS EBS CSI Driver, gp3 StorageClass, IRSA, VPC, Security Groups  
- **Security & Policies:** cert-manager (ACME), Kyverno (admission policies), RBAC  
- **Programming / Scripting:** Python (classroom training + applied automation), Bash  
- **OS & Tools:** Linux (Ubuntu), systemd, kubectl, helm, jq

---

## 📂 Professional Experience

### **Adaequare Info Pvt Ltd – Hyderabad, Telangana**  
**Associate Cloud DevOps Engineer | Apr 2025 – Present**  
**Graduate Engineer Trainee | Feb 2024 – Mar 2025**

- Led design, build, and operationalization of **in-house production-style projects** covering cloud automation and cluster observability while working independently end-to-end.
- Performed platform engineering for Kubernetes (EKS) clusters, secure ingress, monitoring pipelines, and persistent storage with AWS services and K8s CSI.

**Key responsibilities & outcomes**
- Implemented and operated **Kubernetes (EKS)** clusters and worker pools; maintained kubeconfig, node lifecycle, and cluster autoscaling configurations.
- Designed, installed, and configured **NGINX Ingress Controller** and TLS automation via **cert-manager** (Let’s Encrypt) for secure external access to dashboards and apps.
- Built a **hybrid observability stack**:
  - **Prometheus Operator** (in-cluster) for metrics collection and alerting.
  - **Node Exporter**, **cAdvisor**, **kube-state-metrics** for node/pod/container metrics.
  - **Promtail** (DaemonSet) on EKS to forward logs to Bastion-hosted **Loki**.
  - **Grafana** hosted on Bastion for unified dashboards combining Prometheus metrics and Loki logs.
- Deployed **EBS CSI driver** with IRSA and created **gp3 StorageClass** for Prometheus persistent volumes; documented IAM policy and eksctl commands for reproducible setup.
- Implemented **Kyverno ClusterPolicies** to protect critical namespaces and enforced admission controls to prevent accidental deletion of `default`, `kube-system`, `monitoring`, and `adq-dev`.
- Automated verification and validation processes using Python and shell scripts for post-deployment checks, certificate/ingress health, and Prometheus target validation.
- Troubleshot and remediated cluster-level issues: Metrics Server conflicts with EKS add-ons, Prometheus CRD issues, Promtail connectivity to Loki, and NLB/Ingress health checks.
- Produced comprehensive documentation for the platform: installation steps, YAML manifests, runbooks, and troubleshooting guides.

**In-house Projects (selected)**

**AFRA — EKS Monitoring & Observability (In-house project)**  
- Designed and implemented a complete monitoring pipeline for AFRA: Prometheus Operator, ServiceMonitors/PodMonitors, Grafana dashboards, Loki log aggregation, and Promtail DaemonSet.  
- Implemented node-level exporters (Node Exporter & cAdvisor) for system and container metrics and configured Prometheus scrape discovery via ServiceMonitor/PodMonitor resources.  
- Configured secure access to monitoring endpoints (Ingress + cert-manager TLS).  
- Implemented gp3-backed PVCs for Prometheus data retention and created IRSA role for the EBS CSI controller.  
- Authored operational runbooks, Grafana dashboard JSON imports, and automated validation scripts.

**GCP Infrastructure Automation (In-house project)**  
- Automated GCP resource provisioning and configuration using **Terraform** modules and Ansible for post-provisioning tasks.  
- Developed Jenkins pipelines for automated deployment; integrated secrets with GCP Secret Manager and artifact storage in GCS.  
- Implemented monitoring hooks to export metrics to Prometheus and dashboards for visibility.  
- Documentation & project notes: [Notion — GCP Automation Project](https://fortunate-headlight-ff9.notion.site/ADQ-UBUNTUDESKTOP-IAC-8caceafdf6f14875810396b9a136a86e?pvs=143)

---

## 🚀 Major Projects & Documentation
- **Production-Grade EKS Monitoring & Namespace Security (AFRA)** — implemented end-to-end monitoring, logging, TLS, and namespace protection. Documentation: [Medium article — Monitoring Stack on Amazon EKS](https://medium.com/@saravana08052002/building-a-secure-monitoring-stack-on-amazon-eks-tls-prometheus-grafana-promtail-loki-aede23064461)  
- **GCP Infrastructure Automation** — Terraform & Ansible based automation. Notes & playbooks: [Notion Project Notes](https://fortunate-headlight-ff9.notion.site/ADQ-UBUNTUDESKTOP-IAC-8caceafdf6f14875810396b9a136a86e?pvs=143)

---

## 🎓 Education
- **B.E. – Electronics & Communication Engineering** | Panimalar Institute of Technology, Chennai (2019–2023) — CGPA: 8.32  
- **Intermediate (MPC)** | Sri Chaitanya Junior College, Chittoor (2017–2019) — CGPA: 9.57  
- **SSC** | Keshava Reddy E/M High School, Chittoor (2016–2017) — CGPA: 9.2

---

## 🧾 Certifications & Training
- Naresh i Technologies — DevOps Internship Program (2023–2024)  
- Python Programming — Classroom Training (Core Python, Scripting, Automation)  
- Hands-on AWS, EKS, Terraform, Jenkins, and Observability stack labs

---

## 🌐 Languages
English, Telugu, Tamil, Hindi

---

## 🔗 Quick Links (for recruiters / reviewers)
- Medium: Building a secure monitoring stack on Amazon EKS — https://medium.com/@saravana08052002/building-a-secure-monitoring-stack-on-amazon-eks-tls-prometheus-grafana-promtail-loki-aede23064461  
- Notion (GCP project notes): https://fortunate-headlight-ff9.notion.site/ADQ-UBUNTUDESKTOP-IAC-8caceafdf6f14875810396b9a136a86e?pvs=143  
- Monitoring contribution doc (detailed): `MY_CONTRIBUTIONS.md` (attach in repo)

---
