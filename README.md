# Kubernetes GitOps Monorepo

## Overview

This repository acts as the **Single Source of Truth** for my Kubernetes infrastructure. It utilises **GitOps** principles to manage the cluster state, infrastructure, and application lifecycles.

The goal of this project is to demonstrate a production-ready approach to managing Kubernetes clusters using **Infrastructure as Code (IaC)**.

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
├── apps/                          # Application manifestst
│   ├── base/                      # Base Kustomize manifests (DRY principle)
│   └── staging/                   # Application-specific settings
├── clusters/                      
│   └── staging/                
│       └──  flux-system/          # Flux system components
|   ├── .sops.yaml                 # Contains the public key for de
|   ├── apps.yaml        
|   ├── infrastructure.yaml        # Sync definition for infrastructure
│   └── monitoring.yaml            
├── infrastructure/                # Core system components
│   └──  controllers/              
│       └── monitoring/
|           └── base
|               └── renovate
|           └── staging
|               └── renovate
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

# Add the private key to the cluster.
# With this the encrypted secrets are decrypted inside the cluster
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin


```