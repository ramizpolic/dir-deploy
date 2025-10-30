# Directory Deployment

This repository contains the deployment manifests for AGNTCY Directory project.
It is designed to be used with Argo CD for GitOps-style continuous deployment.

The manifests are organized into two main sections:
- `projects/`: Contains Argo CD project definitions.
- `projectapps/`: Contains Argo CD application definitions.

The project will deploy the following components:
- `applications/dir` - AGNTCY Directory server with storage backend
- `applications/dir-admin` - AGNTCY Directory Admin CLI client
- `applications/spire*` - SPIRE stack for identity and federation

**NOTE**: This is not a production-ready deployment. It is
provided as-is for demonstration and testing purposes.

## Onboarding

To onboard a new environment to **Directory Public Staging Network**, check the
[onboarding guide](onboarding/README.md).

## Quick Start

This guide demonstrates how to set up AGNTCY Directory project using
Argo CD in Kubernetes Minikube cluster.

1. Create Minikube cluster

```bash
minikube start
```

2. Install Argo CD in the cluster

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --namespace argocd --for=condition=available deployment --all --timeout=120s
```

3. Deploy Directory via ArgoCD

```bash
# Add project
kubectl apply -f https://raw.githubusercontent.com/ramizpolic/dir-deploy/main/projects/dir/dev/dir-dev.yaml

# Add application
kubectl apply -f https://raw.githubusercontent.com/ramizpolic/dir-deploy/main/projectapps/dir/dev/dir-dev-projectapp.yaml
```

4. Check results in ArgoCD UI

```bash
# Retrieve password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo

# Port forward the ArgoCD API to localhost:8080
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Login to the UI at [https://localhost:8080](https://localhost:8080) with username `admin` and the password retrieved above.

Verify deployment by checking the results of CronJobs in `dir-admin` application.

5. Clean up

```bash
minikube delete
```

### Production Setup

**NOTE:** It is not recommended to deploy both dev and prod environments
in the same cluster, as they may conflict with each other.

If you wish to deploy production-grade setup with
your own domains and Ingress capabilities on top,
follow these steps:

1. Create Minikube cluster

```bash
minikube start
```

2. (optional) Enable Ingress and DNS addons in Minikube

```bash
# Enable Ingress and Ingress-DNS addons
minikube addons enable ingress
minikube addons enable ingress-dns

# Patch Ingress controller to enable SSL Passthrough
kubectl patch deployment -n ingress-nginx ingress-nginx-controller --type='json' \
-p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"--enable-ssl-passthrough"}]'
```

3. (optional) Enable Local DNS inside Minikube

The deployment uses `test` domain for Ingress resources. 

If you want to use local DNS resolution for testing purposes,
follow the guide https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns.

4. (optional) Install CertManager with Self-Signed Issuer

The deployment uses CertManager `letsencrypt` issuer to issue TLS certificates for Ingress resources.

If you want to use self-signed certificates for testing purposes, install CertManager and create a self-signed issuer as follows:

```bash
# Install Cert-Manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.1/cert-manager.yaml

# Wait for Cert-Manager to be ready
kubectl wait --namespace cert-manager --for=condition=available deployment --all --timeout=120s

# Create Self-Signed Issuer and Root CA Certificate
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
   name: selfsigned-issuer
spec:
   selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
   name: my-selfsigned-ca
   namespace: cert-manager
spec:
   isCA: true
   commonName: my-selfsigned-ca
   secretName: root-secret
   privateKey:
      algorithm: ECDSA
      size: 256
   issuerRef:
      name: selfsigned-issuer
      kind: ClusterIssuer
      group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
   name: letsencrypt
spec:
   ca:
      secretName: root-secret
EOF
```

5. Install Argo CD in the cluster

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --namespace argocd --for=condition=available deployment --all --timeout=120s
```

6. Deploy Directory via ArgoCD

```bash
# Add project
kubectl apply -f https://raw.githubusercontent.com/ramizpolic/dir-deploy/main/projects/dir/prod/dir-prod.yaml

# Add application
kubectl apply -f https://raw.githubusercontent.com/ramizpolic/dir-deploy/main/projectapps/dir/prod/dir-prod-projectapp.yaml
```

7. Check results in ArgoCD UI:

```bash
# Retrieve password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo

# Port forward the ArgoCD API to localhost:8080
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Login to the UI at [https://localhost:8080](https://localhost:8080) with username `admin` and the password retrieved above.

Verify deployment by checking the results of CronJobs in `dir-admin` application.

8. Clean up

```bash
minikube delete
```

## Using tokens with local Directory Client

To generate a SPIFFE SVID token for authenticating local Directory Client
with the Directory Server, follow these steps:

1. Create a SPIFFE SVID for local Directory Client

```bash
# For dev environment, same applies for prod environment with appropriate names
kubectl exec spire-dir-dev-argoapp-server-0 -n dir-dev-spire -c spire-server -- \
   /opt/spire/bin/spire-server x509 mint \
   -dns dev.api.directory.outshift.test \
   -spiffeID spiffe://dev.directory.outshift/local-client \
   -output json > spiffe-dev.json
```

2. Set SPIFFE Token variable for Directory Client

```bash
# Set authentication method to token
export DIRECTORY_CLIENT_AUTH_METHOD="token"
export DIRECTORY_CLIENT_SPIFFE_TOKEN="spiffe-dev.json"

# If connecting via Ingress, use:
export DIRECTORY_CLIENT_SERVER_ADDRESS="dev.api.directory.outshift.test:443"

# If connecting via port-forward, use:
export DIRECTORY_CLIENT_SERVER_ADDRESS="127.0.0.1:8888"
```

3. (optional) Port-forward Directory Server to localhost

```bash
# Set server address to localhost
export DIRECTORY_CLIENT_SERVER_ADDRESS="127.0.0.1:8888"

# Skip TLS verification for port-forwarding
export DIRECTORY_CLIENT_TLS_SKIP_VERIFY="true"

# Port-forward
kubectl port-forward svc/dir-dir-dev-argoapp-apiserver -n dir-dev-dir 8888:8888
```

4. Run Directory Client
```bash
dirctl info baeareiesad3lyuacjirp6gxudrzheltwbodtsg7ieqpox36w5j637rchwq
```

## Copyright Notice

[Copyright Notice and License](./LICENSE.md)

Distributed under Apache 2.0 License. See LICENSE for more information.
Copyright AGNTCY Contributors (https://github.com/agntcy)
