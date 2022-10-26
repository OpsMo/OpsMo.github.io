---
title: Create AKS Cluster (az cli) and integrate it with AzureAD user
---

Here, we will show how to create an AKS cluster using the Azure CLI (az command) and integrate it with Azure Active Directory (Azure AD) user authentication.

---

### **- Prerequisites**

Before starting, ensure the following:

1. **Azure CLI**
2. **kubectl** 
3. Required permission to create resources in Azure, including the AKS cluster and Azure AD integration.

---

### **- Create a Resource Group**

A resource group is like a container to hold all the related resources for your AKS cluster.

```bash
az group create --name myaks-rg --location EastUS
```

- **`myaks-rg`** Name of the resource group *(choose you own).*

---

### **- Create the AKS Cluster with Azure AD Integration**

Run the following command to create an AKS cluster that integrates with Azure AD:

```bash
az aks create \
    --resource-group myaks-rg \
    --name myAKSCluster \
    --node-count 1 \
    --enable-aad \
    --enable-managed-identity \
    --generate-ssh-keys
```

### Explanation:

- **`-enable-aad`** for enabling Azure AD integration for user authentication.
- **`-enable-managed-identity`**  to simplifies the management of identities without requiring a service principal.

---

### **- Retrieve Cluster Credentials**

Once the cluster is created, configure your `kubectl` to interact with the AKS cluster using the following command:

```bash
az aks get-credentials --resource-group myaks-rg --name myAKSCluster
```

This sets up your local `kubeconfig` file, allowing you to run `kubectl` commands to manage the cluster.

---

### **- Verify Access**

Test if you can connect to the cluster by listing the nodes:

```bash

kubectl get nodes
```

If the command returns a list of nodes, you’re successfully connected to the AKS cluster.

---

### **- Add Azure AD Users or Groups**

1. **Create Azure AD Groups** (optional):
    
    You can create Azure AD groups for better user management, such as:
    
    ```bash
    az ad group create --display-name "AKS-Admins" --mail-nickname "AKS-Admins"
    az ad group create --display-name "AKS-Readers" --mail-nickname "AKS-Readers"
    
    ```
    
2. **Assign Azure AD Groups to Kubernetes Roles**:
For example, assign the **AKS-Admins** group full admin access:
    
    ```yaml
    yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: aks-admin-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: Group
        name: AKS-Admins # Name of your Azure AD group
        apiGroup: rbac.authorization.k8s.io
    
    ```
    
    Save this as `aks-admin-binding.yaml` and apply it:
    
    ```bash
    kubectl apply -f aks-admin-binding.yaml
    ```
    
3. **Add Users to Azure AD Groups**:
Add Azure AD users to the groups:
    
    ```bash
    az ad group member add --group "AKS-Admins" --member-id "<userObjectId>"
    ```
    
    Replace `<userObjectId>` with the object ID of the user you want to add.
