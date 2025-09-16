# Troubleshooting: Metrics Server Installation on EKS

## Issue Faced

While installing **Metrics Server** on an EKS cluster using Helm, the following error occurred:

```bash
Error: Unable to continue with install: ClusterRole "system:metrics-server-aggregated-reader" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; label validation error: key "app.kubernetes.io/managed-by" must equal "Helm": current value is "EKS"; annotation validation error: missing key "meta.helm.sh/release-name": must be set to "custom-metrics"; annotation validation error: missing key "meta.helm.sh/release-namespace": must be set to "kube-system"
```

### Root Cause

This error occurred because the Metrics Server components were already provisioned by **EKS Managed Add-ons**, and Helm could not take ownership of the existing ClusterRole/ClusterRoleBinding due to mismatched labels and annotations.

## Troubleshooting Steps

1. **Checked existing API Services:**
   ```bash
   kubectl get apiservice | grep metrics
   ```
   Found:
   ```bash
   v1.metrics.eks.amazonaws.com   kube-system/eks-extension-metrics-api   True
   ```

2. **Attempted Helm install (failed due to existing roles).**

3. **Deleted the conflicting ClusterRole:**
   ```bash
   kubectl delete clusterrole system:metrics-server-aggregated-reader
   kubectl delete clusterrolebinding system:metrics-server
   ```

   Output:
   ```bash
   clusterrole.rbac.authorization.k8s.io "system:metrics-server-aggregated-reader" deleted
   Error from server (NotFound): clusterrolebindings.rbac.authorization.k8s.io "system:metrics-server" not found
   ```

4. **Verified remaining roles/bindings:**
   ```bash
   kubectl get clusterrole | grep metrics
   kubectl get clusterrolebinding | grep metrics
   ```

   Found only EKS-managed roles like:
   ```bash
   eks:extension-metrics-apiserver
   eks:k8s-metrics
   ```

5. **Re-installed Metrics Server via Helm:**
   ```bash
   helm upgrade --install custom-metrics metrics-server/metrics-server      --namespace kube-system      --set args={--kubelet-insecure-tls}
   ```

   ✅ Successful deployment with:
   ```bash
   NAME: custom-metrics
   NAMESPACE: kube-system
   STATUS: deployed
   REVISION: 1
   ```

## Final Notes

- The issue was due to conflicting ownership between **EKS add-ons** and **Helm-installed Metrics Server**.  
- Resolution involved removing the pre-existing ClusterRole and re-installing Metrics Server using Helm.  
- Always check if EKS already provides Metrics Server as an **add-on** before manually installing.

