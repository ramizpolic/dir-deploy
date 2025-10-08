# dir-deploy

Directory Public Deployment

## Quick Start

This guide demonstrates how to set up AGNTCY Directory project using
Argo CD in Kubernetes Kind cluster.

1. Create Kind cluster:

   ```bash
   kind create cluster --name argo-dev
   ``` 

2. Install Argo CD in the cluster:

   ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

3. Add Directory ArgoCD deployment:

    ```bash
    # Add project
    kubectl apply -f https://raw.githubusercontent.com/ramizpolic/dir-deploy/main/projects/dir/appproject.yaml

    # Add application
    kubectl apply -f https://raw.githubusercontent.com/ramizpolic/dir-deploy/main/projectapps/dir/application.yaml
    ```


4. Check results in ArgoCD UI:

   ```bash
   # Port forward the ArgoCD API to localhost:8080
   kubectl port-forward svc/argocd-server -n argocd 8080:443

   # Retrieve password
   argocd admin initial-password -n argocd
   ```

   Login to the UI at [https://localhost:8080](https://localhost:8080) with username `admin` and the password retrieved above.

5. Clean up

   ```bash
   kind delete cluster --name argo-dev
   ```
