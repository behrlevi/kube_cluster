# Kubernetes GitOps Monorepo

## Overview

This repository acts as the **Single Source of Truth** for my Kubernetes infrastructure. It utilises **GitOps** principles to manage the cluster state, infrastructure, and application lifecycles.

The goal of this project is to demonstrate a production-ready approach to managing Kubernetes clusters using **Infrastructure as Code (IaC)**.

The project is aligned with [The Twelve-Factor App](https://12factor.net/) methodology.
---

## Core Technology Stack

This project leverages the following technologies to ensure scalability, security, and automation:

| Category | Technology | Usage |
| :--- | :--- | :--- |
| **Orchestration** | ![Kubernetes](https://img.shields.io/badge/-Kubernetes-326ce5?logo=kubernetes&logoColor=white) | Container orchestration and management. |
| **GitOps** | ![Flux](https://img.shields.io/badge/-FluxCD-2962FF?logo=flux&logoColor=white) | Continuous Delivery and cluster state reconciliation. |
| **IaC** | ![Terraform](https://img.shields.io/badge/-Terraform-623CE4?logo=terraform&logoColor=white) | Provisioning underlying cloud/virtual resources. |
| **OS** | ![Linux](https://img.shields.io/badge/-Linux-FCC624?logo=linux&logoColor=black) | Base operating system for nodes. |
| **Scripting** | ![Python](https://img.shields.io/badge/-Python-3670A0?logo=python&logoColor=white) | Automation scripts and custom controllers. |
| **Observability** | **Prometheus & Grafana** | Full stack monitoring, alerting, and metrics visualization. |
| **Networking** | **Ingress-Nginx & MetalLB** | Layer 7 traffic management and Layer 2 Load Balancing. |
| **Security** | **SOPS & Network Policies** | Secret encryption at rest and pod-to-pod traffic isolation. |

---

## Architecture & Workflow

Changes to the infrastructure are performed via Pull Requests. Once merged, Flux automatically synchronises the cluster state with the Git repository.

```mermaid
graph LR
    A[User / CI] -- Commits Code --> B(Git Repository);
    B -- Webhook Trigger --> C{Flux Controller};
    C -- Reconciles State --> D[Kubernetes Cluster];
    
    subgraph Cluster
    D -- Deploys --> E[Apps];
    D -- Configures --> F[Network Policies];
    end
    
    G[Renovate Bot] -- Auto-Updates Dependencies --> B;
    H[Prometheus] -- Scrapes Metrics --> D;
```
## Repository Structure

The repository is structured in conformity with the [monorepo](https://fluxcd.io/flux/guides/repository-structure/) methodology.
```
├── apps/                                             # Application manifestst
│   ├── base/                                         # Base Kustomize manifests
│   └── staging/                                      # Application-specific settings
├── clusters/                      
│   └── staging/                
│       └──  flux-system/                             # Flux system components
            ├── gotk-components.yaml                  # GitOps toolkit
            ├── gotk-sync.yaml                        # GitOps toolkit - kind: GitRepository & kind: Kustomization (the Kustomization file below)
            └── kustomization.yaml                    # The base template for Flux configured by default.
|   ├── .sops.yaml                                    # Contains the public key for encryption
|   ├── apps.yaml                                     # A Kustomization custom resource
|   ├── infrastructure.yaml                           # Sync definition for infrastructure
│   └── monitoring.yaml            
├── infrastructure/                                   # Core system components
│   └──  controllers/              
│       └── monitoring/
|           └── base
|               └── renovate
|           └── staging
|               └── renovate
|                   ├── configmap.yaml                # Credentials for the Renovate Bot
|                   ├── cronjob.yaml                  # Defines a job wich runs at a given interval and checking for image updates
|                   ├── namespace.yaml
|                   └──renovate-cotainer-env.yaml
├── monitoring/
|   └── confings/staging                  
|       └── kube-prometheus-stack         
|           └── grafana-tls-secret.yaml    
|   ├── base
|       ├── namespace.yaml
|       ├── release.yaml                              # Defines the HELM release
|       └── repository.yaml                           # Defines the repository containing the Helm chart
├──renovate.json                                      # Matches all the YAML files in the repository. 
```
## Secrets management with [SOPS](https://fluxcd.io/flux/guides/mozilla-sops/)

#### Encrypting secrets using age
```
brew install sops age

# Creates the public and private keys
# The private key should be stored separately in case
# the cluster has to be rebuilt in the future.
age-keygen -o age.agekey

# Export the pubkey into a variable
export AGE_PUBLIC=<public_key>

# Encrypt the yaml definition of the secret with the pubkey
sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place secret.yaml

# Add the **private key** to the cluster.
# With this the encrypted secrets are decrypted inside the cluster
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```

## Flux
#### Installing Flux
Flux needs to be installed on the cluster to facilitate the GitOps reconciliation loop

Bootstrapping Flux on the Cluster

```
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>

# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Check if the cluster meets the requirements
# Kubernetes 1.28.0.0 or newer is required
flux check --pre

flux bootstrap github \

  --owner=$GITHUB_USER \

  --repository=<repo_name> \

  --branch=main \

  --path=./clusters/staging \

  --personal
```

This installs Flux controllers in a separate namepsace and commits manifests to the specified repository.
Flux creates necessary manifests in the GitHub repository.

#### What is a Kustomization?
A **Kustomization** custom resource represents a local set of Kubernetes resources (e.g. kustomize overlay) the Flux is supposed to reconcile in the cluster.
The reconciliation runs every five minutes by default which can be changed by **.spec.interval**.

#### How does Flux deploy the application?
  1. Flux reads the clusters/staging directory

  2. Finds and applies our apps.yaml file

  3. Looks at the apps/staging directory

  4. Finds the applications kustomization file

  5. Applies the resources we defined in the base configuration

## Configuring Cloudflare Tunnels [manual](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/create-local-tunnel/)
#### Installation of cloudflared

```
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-public-v2.gpg | sudo tee /usr/share/keyrings/cloudflare-public-v2.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-public-v2.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install cloudflared
```
#### Sign in to Cloudflare and authorize the tunnel for a domain
```
cloudflared tunnel login
```
After successful authorization, ~/.cloudflared/cert.pem will be created.

#### Crete a tunnel
```
cloudflared tunnel create <tunnel_name>
```
This creates a .json file containing the *account tag*, *tunnel secret* and the *tunnel ID* in the ~/.cloudflared/ directory.

#### Create a secret resource in Kubernetes to store the tunnel secret

The contents of the secret are loaded from the .json created above. 

```
kubectl create secret generic tunnel-credentials \

--from-file=credentials.json=<ID>.json --namespace=<app_namespace>
```

#### Create a CNAME DNS record
```
<tunnel_ID>.cfargotunnel.com
```
#### Define a Service

```
apiVersion: v1
kind: Service
metadata:
  name: <app>
spec:
  ports:
    - port: 9090
  selector:
    app: <app>
  type: ClusterIP
  ```
#### Create the cloudflared pod

I have defined the pod's configuration in the apps/staging/application.
The contents of the secret containing the tunnel credentials are mounted as a file inside the pod.
I have included a configmap in the cloudflared config manifest.

The logic in the file is as follows:
- 'creds' mounts the 'tunnel-credentials' secret as a file into the pod
- 'config' mounts config.yaml which is created from a ConfigMap resource
- The ConfigMap called cloudflared defines the following parameters:
  - the name of tunnel which has to indentical with the name given in the 'cloudflared tunnel create' command
  - defines the path of the credentials file which was mounted by the 'creds' volume mount and stems from the secret
  - the Ingress hostname and the DNS name of the service to which the tunnel should be bound to

## Installing the Kube-Prometheus-Stack

#### Install Helm cli
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash
```

The manifests are located in the **monitoring** folder.

#### release.yaml
Installs the Helm chart.
The analogous command would be:
```
helm install [RELEASE_NAME] oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack
```
The values section defines the Helm values, which can be printed with the command:
```
helm show values prometheus/community/kube-prometheus-stack | vim
```

## Deploying secrets with Flux

To deploy secrets with Flux, the secrets need to be defined in YAML manifests.
By storing credentials in secrets this way, we are satisfying the [third requirement](https://12factor.net/config) of The Twelve-Factor app methodology. 

The following commands will create a manifest for the secret containing the Linkding Cloudflare tunnel credentials and encrypt it with the AGE key:
```
kubectl create secret generic tunnel-credentials \
--from-file=credentials.json=<tunnel-IDa>.json --dry-run=client -o yaml > cloudflare-secret.yaml

sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place cloudflare-secret.yaml
```

We can also encrypt the environment variables into a secret.
These command create and encrypt the superuser credentials used for Linkding, which doesn't included a built-in user by default.

```
kubectl create secret generic linkding-container-env \
--from-literal=LD_SUPERUSER_NAME=admin \
--from-literal=LD_SUPERUSER_PASSWORD=password \
--dry-run=client \
-o yaml > linkding-container-env-secret.yaml

sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place linkding-container-env-secret.yaml
```

## Automatic updates with Renovate bot

[Renovate](https://github.com/renovatebot/renovate) is an automated dependecy update tool.
The goal of it's usage in the current project is to have a system in place running 24/7 which periodically checks for new image versions
and creates a GitHub pull request when an update is available. The user can decide to update the given image by approving the PR.

Requirements:
- a GitHub token with **repo** permission

#### Create and encrypt a secret containing the GitHub token
```
kubectl create secret generic renovate-container-env \
--from-literal=RENOVATE_TOKEN=<token> \
--dry-run=client \
-o yaml > renovate-container-env.yaml

sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place renovate-container-env.yaml
```
#### Manifest files
The YAML files are placed in the infrastructure/controllers/base/renovate directory.