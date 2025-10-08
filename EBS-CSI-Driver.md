# AWS EBS CSI Driver and gp3 StorageClass Setup for EKS – Complete Guide

---

## 🧩 Introduction

The **AWS EBS CSI Driver** enables Amazon Elastic Kubernetes Service (EKS) clusters to manage **Elastic Block Store (EBS)** volumes as persistent storage for pods. It is essential for stateful workloads like databases, Prometheus, and Jenkins.

### Why use the AWS EBS CSI Driver?

* The in-tree EBS plugin is deprecated — CSI is the **modern, modular, and supported approach**.
* Provides dynamic volume provisioning using AWS EBS volumes.
* Supports volume resizing and snapshots.

### Why use gp3 over gp2?

* **gp3** offers better performance and cost efficiency.
* Allows independent configuration of IOPS and throughput.
* Default in modern EKS clusters.

### Why IRSA (IAM Role for Service Account)?

* IRSA ensures **fine-grained IAM permissions** for Kubernetes pods.
* The EBS CSI driver runs with the necessary permissions without using node-level IAM roles.

---

## ⚙️ Step 1: Prerequisites

Ensure you have the following tools installed:

```bash
aws --version
kubectl version --client
helm version
eksctl version
```

Verify your cluster and region:

```bash
aws eks list-clusters --region us-east-1
kubectl get nodes
```

---

## 🪪 Step 2: Associate OIDC Provider for IRSA

Run the following command to associate your EKS cluster with an IAM OIDC provider (only required once per cluster):

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster adq-dev-eks \
  --approve
```

---

## 🧱 Step 3: Create IAM Policy for the EBS CSI Driver

Create the IAM policy required by the EBS CSI driver.

### Create the Policy JSON

Save the following as `AmazonEKS_EBS_CSI_Driver_Policy.json`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateSnapshot",
                "ec2:AttachVolume",
                "ec2:DetachVolume",
                "ec2:ModifyVolume",
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeInstances",
                "ec2:DescribeSnapshots",
                "ec2:DescribeTags",
                "ec2:DescribeVolumes",
                "ec2:CreateTags",
                "ec2:DeleteTags",
                "ec2:CreateVolume",
                "ec2:DeleteVolume"
            ],
            "Resource": "*"
        }
    ]
}
```

### Create the IAM Policy in AWS

```bash
aws iam create-policy \
  --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
  --policy-document file://AmazonEKS_EBS_CSI_Driver_Policy.json
```

Note the ARN returned — you will use it in the next step.

---

## 🔐 Step 4: Create IAM Role and Kubernetes Service Account

Use **eksctl** to create a service account with the IAM role attached.

```bash
eksctl create iamserviceaccount \
  --region us-east-1 \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster adq-dev-eks \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKS_EBS_CSI_Driver_Policy \
  --approve \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

This will automatically:

* Create an IAM role named **AmazonEKS_EBS_CSI_DriverRole**.
* Annotate the service account `ebs-csi-controller-sa` with the role.

---

## 🚀 Step 5: Install the AWS EBS CSI Driver using Helm

Add the official Helm repo:

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```

Install the driver:

```bash
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=ebs-csi-controller-sa
```

Verify installation:

```bash
kubectl get pods -n kube-system | grep ebs
```

✅ Expected output:

```
ebs-csi-controller-xxxxxxx   5/5   Running   0   2m
ebs-csi-node-xxxxx           3/3   Running   0   2m
```

---

## 📦 Step 6: Create the gp3 StorageClass

Save this YAML as `storageclass.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Apply it:

```bash
kubectl apply -f storageclass.yaml
```

Verify:

```bash
kubectl get sc
```

✅ Expected output:

```
NAME   PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp3    ebs.csi.aws.com   Delete          WaitForFirstConsumer   false                  10s
gp2    kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false              14d
```

---

## 🧩 Step 7: Validation

Check if PVCs are dynamically provisioned:

```bash
kubectl get pvc -A
kubectl describe pvc <pvc-name>
```

You should see a volume provisioned using the **gp3** class.

Verify EBS CSI controller and node status:

```bash
kubectl get pods -n kube-system | grep ebs
```

If all pods are running — the setup is complete ✅

---

## 🩺 Troubleshooting

### ❌ Error: `no EC2 IMDS role found`

**Reason:** The EBS CSI driver pods can’t assume an IAM role.
**Fix:**

* Ensure IRSA is enabled (`eksctl utils associate-iam-oidc-provider`)
* Ensure the IAM role has the trust policy with OIDC provider.

### ❌ PVC stuck in `Pending`

**Reason:** CSI driver not running or wrong provisioner.
**Fix:**

* Check pods: `kubectl get pods -n kube-system | grep ebs`
* Check storage class provisioner → must be `ebs.csi.aws.com`.

---

## ✅ Final Verification

Once all configurations are correct:

```bash
kubectl get sc
kubectl get pods -n kube-system | grep ebs
kubectl describe pvc <pvc-name>
```

✅ All EBS CSI controller and node pods should be **Running**.
✅ New PVCs should dynamically create **EBS gp3** volumes in AWS.
✅ Storage provisioning is fully automated.

---

### 🧠 Summary

| Component            | Purpose                              | Notes                       |
| -------------------- | ------------------------------------ | --------------------------- |
| **EBS CSI Driver**   | Attaches and manages AWS EBS volumes | Runs in kube-system         |
| **IRSA**             | Grants IAM permissions to CSI driver | Secure and least privilege  |
| **gp3 StorageClass** | Default storage class for workloads  | Faster and cheaper than gp2 |

---

**🎯 Your EKS cluster now supports dynamic EBS gp3 volume provisioning using the AWS EBS CSI Driver!**
