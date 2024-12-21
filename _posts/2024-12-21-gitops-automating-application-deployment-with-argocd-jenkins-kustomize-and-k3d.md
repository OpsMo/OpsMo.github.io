---
title: 'GitOps: Automating Application Deployment with ArgoCD, Jenkins, Kustomize,
  and K3d'
---

![image info](assets/images/Jenkins-K3d-argo-K.png){: width="550" }

Here, we'll explore how to set up a GitOps workflow using ArgoCD, Jenkins, Kustomize, and K3d. This setup enables automated application deployment to Kubernetes environments with seamless management of multiple environments like `dev` and `prod`.

The accompanying repository can be found here: [argocd-app](https://github.com/OpsMo/argocd-app).

---

## Repository Structure

The repository is organized as follows:

- **app-ci-image/**: Contains Dockerfile and Jenkins pipeline (`Jenkinsfile`) to build and push the application image to DockerHub.
- **k8s-app-kustomize/**: Contains Kustomize configurations for managing Kubernetes resources.
    - **common/**: Base resources shared across environments.
    - **environments/**: Overlays for specific environments (`dev`, `prod`).
- **argocd-application-dev.yaml**: Defines the ArgoCD application configuration for the development environment.
- **argocd-application-prod.yaml**: Defines the ArgoCD application configuration for the production environment.

---

## Prerequisites

To follow along, ensure you have the following:

- A Kubernetes cluster (e.g., K3d for local development).
- ArgoCD installed in the cluster.
- `kubectl` CLI installed locally.

---

## Setting Up the Environment

### 1. Clone the Repository

Start by cloning the repository:

```bash
git clone https://github.com/OpsMo/argocd-app.git
cd argocd-app
```

### 2. Build and Push the Application Image

### Option 1: Manual Build

Navigate to the `app-ci-image` directory:

```bash
cd app-ci-image
```

Build and push the Docker image:

```bash
docker build -t your-repo/your-app:latest .
docker push your-repo/your-app:latest

```

### Option 2: Jenkins Pipeline

Use the provided Jenkins pipeline to automate the process. The `Jenkinsfile` in the `app-ci-image` directory is preconfigured for this purpose:

- Configure Jenkins to use the pipeline script from SCM.
- Trigger the pipeline to build and push the image to DockerHub.

---

## Kustomize: Managing Multiple Environments

### Common Directory (`k8s-app-kustomize/common`)

The `common` directory contains base resources that are shared across environments:

- **deployment.yaml**: Defines the base deployment for Nginx.
- **nginx-service.yaml**: ClusterIP service for Nginx.
- **nginx-ingress.yaml**: Configures ingress for routing traffic.
- **kustomization.yaml**: Lists the resources included in the base configuration.

Example `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- nginx-service.yaml
- nginx-ingress.yaml

```

### Environment-Specific Overlays (`k8s-app-kustomize/environments`)

Each environment (e.g., `dev`, `prod`) contains specific configurations:

### Dev Directory

- **kustomization.yaml**: Includes the base resources and applies patches.
- **deployment-dev.yaml**: Adds environment variables for `dev`.
- **ingress-dev.yaml**: Customizes ingress paths for `dev`.

Example `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../common
patches:
  - target:
      kind: Ingress
      name: nginx-ingress
    path: ingress-dev.yaml
  - target:
      kind: Deployment
      name: nginx
    path: deployment-dev.yaml
namespace: dev

```

---

## Configuring ArgoCD Application

The repository includes ArgoCD application manifests for `dev` and `prod` environments. Below is an example configuration for `dev`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
  namespace: argocd
spec:
  destination:
    namespace: dev
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://github.com/OpsMo/argocd-app.git
    targetRevision: main
    path: k8s-app-kustomize/environments/dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```

To deploy:

```bash
kubectl apply -f argocd-application-dev.yaml -n argocd
```

---

## Rollback with ArgoCD

Rollback to a previous version using ArgoCD CLI:

1. View the application history:
    
    ```bash
    argocd app history my-app-dev
    ```
    
2. Rollback to a specific revision:
    
    ```bash
    argocd app rollback my-app-dev --revision <revision-number>
    ```
    
3. Verify the rollback:
    
    ```bash
    argocd app get my-app-dev
    ```
    

---

## Best Practices

- **GitOps Principles**: Store all configurations in version control (Git) and use ArgoCD for syncing changes.
- **Separate Environments**: Manage `dev` and `prod` using Kustomize overlays.
- **Automate CI/CD**: Leverage Jenkins (or any other CI tools like Github actions) for consistent and automated image/artifact builds.

---

## Explore the Repository

Follow the repository to explore the complete project structure and configurations: [argocd-app](https://github.com/OpsMo/argocd-app).

Goodluck
