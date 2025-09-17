# Kyverno Setup in EKS (adq-dev-eks)

## 🔎 What is Kyverno?
Kyverno is a Kubernetes-native policy engine that allows you to **validate, mutate, and generate** configurations using Kubernetes-style YAML.  
It works as an **admission controller**, intercepting requests to the Kubernetes API server and applying defined rules.

- Website: [https://kyverno.io](https://kyverno.io)
- Helm charts: [https://kyverno.github.io/kyverno](https://kyverno.github.io/kyverno)

## 🎯 Why Kyverno?
In our EKS cluster (`adq-dev-eks`), we need to **protect critical namespaces** like `default`, `kube-system`, and custom namespaces from accidental deletion.  
Kyverno policies enforce this protection at the cluster level, preventing disruptive actions.

---

## 🚀 Steps Performed

### 1. Connect to EKS Cluster
From the Bastion host:
```bash
aws eks update-kubeconfig --region ap-south-1 --name adq-dev-eks
kubectl get nodes
```

### 2. Install Kyverno
```bash
helm repo add kyverno https://kyverno.github.io/kyverno
helm repo update
helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace
kubectl -n kyverno get pods
```
> All Kyverno pods should be `Running`.

### 3. Create the Protect Namespaces Policy
File: **`protect-namespaces.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: protect-namespaces
spec:
  validationFailureAction: Enforce
  rules:
    - name: prevent-deleting-namespaces
      match:
        resources:
          kinds:
            - Namespace
      validate:
        message: "Deletion of protected namespaces is not allowed."
        deny:
          conditions:
            all:
              - key: "{{request.operation}}"
                operator: Equals
                value: "DELETE"
              - key: "{{ request.name || request.object.metadata.name }}"
                operator: In
                value:
                  - "default"
                  - "kube-system"
                  - "kube-public"
                  - "kube-node-lease"
                  - "namespace1"
                  - "namespace2"
```

Apply:
```bash
kubectl apply -f protect-namespaces.yaml
```

### 4. Verify Policy
```bash
kubectl get clusterpolicy protect-namespaces -o yaml
```

### 5. Test Policy
```bash
kubectl delete namespace namespace1
```
**Expected Result:**
```
Error from server: admission webhook "validate.kyverno.svc" denied the request:
Deletion of protected namespaces is not allowed.
```

---

## 🔓 Emergency/Unblock Options

### Option 1: Edit Policy
Remove the namespace from the protected list:
```bash
kubectl edit clusterpolicy protect-namespaces
```
Then retry the deletion:
```bash
kubectl delete namespace <namespace-name>
```

### Option 2: Delete the Policy
Completely remove the protection:
```bash
kubectl delete clusterpolicy protect-namespaces
kubectl delete namespace <namespace-name>
```
Re-apply later to restore:
```bash
kubectl apply -f protect-namespaces.yaml
```

---

## ✅ Summary
- Installed Kyverno on `adq-dev-eks`
- Created a **ClusterPolicy** to prevent deletion of critical namespaces
- Verified enforcement with a test namespace
- Documented emergency override options

Kyverno ensures **operational safety** by protecting important namespaces and preventing human error in production clusters.
