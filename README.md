# GitOps Project

GitOps repository for deploying and managing the Container App on Kubernetes using Helm and Argo CD. It supports separate development, QA, and production environments, with automated synchronization between Git and the Kubernetes cluster.

## Project Overview

Project objectives:

* Create a GitOps repository.
* Store Helm charts and environment-specific configuration.
* Use Argo CD ApplicationSet to generate applications for multiple environments.
* Automate application deployment and updates using GitOps.
* Document how to install, configure, and access Argo CD.

## Project Structure

```bash
.
├── applicationsets/
│   └── env-applicationset.yaml
├── helmchart/
│   ├── Chart.yaml               
│   ├── values.yaml              
│   ├── templates/
│   │   ├── configmap.yaml       
│   │   ├── deployment.yaml      
│   │   ├── ingress.yaml         
│   │   └── service.yaml         
│   └── env/
│       ├── dev/
│       │   └── values.yaml      
│       ├── qa/
│       │   └── values.yaml      
│       └── prd/
│           └── values.yaml     
└── README.md                    
```

## Environment Differences

Each environment uses the same Helm chart but has its own `values.yaml` file.

| Environment | Namespace | Replicas |
|---|---|---:|
| Development | `dev` | 1 |
| QA | `qa` | 2 |
| Production | `prd` | 3 |

The replica counts are configured independently in each environment's values file. Other settings can also be customized per environment.

## Prerequisites

- Git
- Kubernetes cluster (Minikube or another supported cluster)
- kubectl`
- Helm
- Argo CD
- Access to the GitOps repository

## Deployment Setup

### 1. Clone the GitOps repository

```bash
git clone https://github.com/LizaSaitov/GitOps-Project.git
cd GitOps-Project
```

### 2. Start the Kubernetes cluster

**Local machine or cloud shell using Minikube**

Install Minikube, `kubectl`, and Helm, then start the cluster:

```bash
minikube start
kubectl get nodes
```

### 3. Install Argo CD

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD using Helm:

```bash
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo-cd/argo-cd \
  --namespace argocd
```

Check the Argo CD pods:

```bash
kubectl get pods -n argocd
```

Wait until the required pods are running before continuing.

### 4. Apply the ApplicationSet

From the root directory of the GitOps repository, run:

```bash
kubectl apply -f applicationsets/env-applicationset.yaml
```

The ApplicationSet uses a Git directory generator to discover the environment directories under `helmchart/env/`. It generates an Argo CD Application for each environment and selects the corresponding Helm values file.

The generated applications deploy to the `dev`, `qa`, and `prd` namespaces. Automated synchronization, pruning, and self-healing are configured in the ApplicationSet.

### 5. Access the Argo CD web interface

Start a port-forward in a terminal:

```bash
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443
```

Keep this terminal running while accessing the interface.

**Local machine:** Open `https://localhost:8080` in your browser.

**Google Cloud Shell:** Use Cloud Shell's Web Preview feature to open port `8080`. If necessary, configure the port-forward and browser access according to your Cloud Shell environment.

Retrieve the initial administrator password:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

Log in with username `admin` and the retrieved password.

### 6. Verify the deployments

Check the generated Argo CD applications:

```bash
kubectl get applications -n argocd
```

Check the deployments in each environment:

```bash
kubectl get deployments -n dev
kubectl get deployments -n qa
kubectl get deployments -n prd
```

The applications should synchronize with the GitOps repository, and the deployments should reach their desired replica counts.

## Deploying and Updating Applications

Application configuration is managed through the GitOps repository.

1. Update the relevant environment's `values.yaml` file or modify the Helm chart.
2. Commit and push the changes to the GitOps repository. Changes can be pushed by the repository owner or another user with write access.
3. Argo CD detects changes in the Git repository and automatically synchronizes the affected applications.
4. Kubernetes applies the updated Kubernetes resources, and Kubernetes controllers work to maintain the desired state.

For example, to change the development replica count, update:

```
helmchart/env/dev/values.yaml
```

Commit and push the change:

```bash
git add helmchart/env/dev/values.yaml
git commit -m "Update development replicas"
git push origin main
```

Argo CD then reconciles the development application with the updated configuration.
