---
title: Deploying ArgoCD on k3d for Local Development (macOS)
tags:
- Kubernetes
- k3d
- GitOps
- ArgoCD
- Traefik
- Ingress
---

Here, we’ll deploy **Argo CD** in a local Kubernetes cluster (On MacOs) using **k3d** and expose it using **Traefik** Ingress on the subpath /argo. By the end, you’ll be able to access Argo CD at:

http://localhost:8080/argo

*So, before we start:*

***Why use Ingress instead of `kubectl port-forward`?***

*While it's possible to expose Argo CD using kubectl port-forward, I chose Ingress to provide a more robust and user-friendly way to access the service. This setup better reflects real-world scenarios and makes it easier to manage multiple services.*

*Note:*

*The k3d cluster comes with Traefik as the default ingress controller, which makes it convenient for this Test. However, you’re free to replace it with your preferred ingress controller if you want.*

**1.Installing K3d (as lightweight Kubernetes for local development)**

```bash
brew install k3d 
```

and install kubectl client (if needed)

```bash
brew install kubectl
```

## **2. Create a Fresh k3d Cluster**

Create a k3d cluster with Traefik’s load balancer exposed on port 8080:

```bash
k3d cluster create myk8s --port "8080:80@loadbalancer"
```

Verify the cluster:

```bash
kubectl cluster-info
```

---

## **3. Install Argo CD**

1. **Create the Argo CD Namespace**:
    
    ```bash
    kubectl create namespace argocd
    ```
    
2. **Download the Official Argo CD Manifest**:

```bash
wget https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -O argocd-install.yaml
```

1. **Edit the Argo CD Server Configuration**:
    
    To expose Argo CD at /argo and disable https for local testing.
    Open the `argocd-install.yaml` file and locate the **`argocd-server`** container.
    
    Add the following `args` section under the `containers` field:
    
    ```yaml
    args:
      - "argocd-server"
      - "--insecure"
      - "--rootpath=/argo"
      - "--basehref=/argo"
    ```
    
    The full container definition will look like this:
    
    ```yaml
    containers:
      - name: argocd-server
        image: quay.io/argoproj/argocd:latest
        args:
          - "argocd-server"
          - "--insecure"
          - "--rootpath=/argo"
          - "--basehref=/argo"
        ports:
          - containerPort: 80
            name: http
          - containerPort: 443
            name: https
    ```
    
2. **Apply the Modified Argo CD Manifest**:
    
    ```bash
    kubectl apply -f argocd-install.yaml -n argocd
    ```
    
3. **Verify Argo CD Deployment**:
    
    ```bash
    kubectl get pods -n argocd
    ```
    

All pods should eventually show `Running` status.

---

## **4. Configure Ingress for Argo CD**

To expose Argo CD at `/argo` on `http://localhost:8080`, create an Ingress resource.

1. **Create `argocd-ingress.yaml`**:
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: argocd-ingress
      namespace: argocd
      annotations:
        kubernetes.io/ingress.class: traefik
        traefik.ingress.kubernetes.io/ssl-redirect: "false"
    spec:
      rules:
        - host: localhost
          http:
            paths:
              - path: /argo
                pathType: Prefix
                backend:
                  service:
                    name: argocd-server
                    port:
                      number: 80
    ```
    
2. **Apply the Ingress**:
    
    ```bash
    kubectl apply -f argocd-ingress.yaml
    ```
    
3. **Verify the Ingress**:
    
    ```bash
    kubectl get ingress -n argocd
    ```
    

---

## **5. Access ArgoCD UI**

1. Retrieve the Argo CD **admin password**:
    
    ```bash
    kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
    ```
    
2. Open your browser and navigate to:
    
    ```bash
    http://localhost:8080/argo
    ```
    
3. Log in with:
    - **Username**: `admin`
    - **Password**: Use the password retrieved above.

---

## **6. Troubleshooting**

1. **Argo CD UI Redirects Incorrectly**
    
    Ensure the `--rootpath` and `--basehref` arguments are both set in `argocd-server`.
    
2. **Pods Not Starting**
    
    Verify the Argo CD manifest only modifies the `argocd-server` container and leaves other components untouched.
    
3. **Check Logs**
    
    Use the following to debug any issues:
    
    ```bash
    kubectl logs -l app.kubernetes.io/name=argocd-server -n argocd
    ```
    

---

This setup is ideal for testing and local development workflows.
Good Luck
