# dir-deploy

Directory Public Deployment


## Quick Start

1. Create Kind cluster:

   ```bash
   kind create cluster --name argo-dev
   ``` 

2. Install Argo CD in the cluster:

   ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

3. Configure Argo CD:

   ```bash
   argocd cluster add <context-name>
   ```

4. Add Directory ArgoCD deployment:

    ```bash
    kubectl apply -f applicationsets/dir/dev/applicationset.yaml
    ```

5. Clean up

   ```bash
   kind delete cluster --name argo-dev
   ```
