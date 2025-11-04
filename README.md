# Keycloak HA Deployment with ArgoCD

This repository contains the manifests for deploying a High Availability Keycloak setup with PostgreSQL database managed by Zalando operator on Kubernetes using ArgoCD.

## Components

- **PostgreSQL Cluster**: 2-replica PostgreSQL cluster using Zalando operator
- **Keycloak**: 3-replica StatefulSet with JGroups clustering for HA
- **SOPS Encryption**: Database credentials encrypted with SOPS
- **ArgoCD**: GitOps deployment automation

## Repository Structure

```
.
├── .sops.yaml                    # SOPS configuration
├── postgres-cluster.yaml         # PostgreSQL cluster definition
├── keycloak-db-secret.yaml       # SOPS-encrypted database credentials
├── keycloak-deployment.yaml      # Keycloak StatefulSet and Services
├── keycloak-ingress.yaml         # Keycloak Ingress
├── argocd-application.yaml       # ArgoCD Application manifest
└── README.md                     # This file
```

## Prerequisites

1. Kubernetes cluster with kubectl access
2. Zalando PostgreSQL Operator installed
3. ArgoCD installed with SOPS support (GPG key configured)
4. Cert-manager with rancher-subca ClusterIssuer
5. Nginx Ingress Controller

## Setup Instructions

### 1. Create GitHub Repository

First, create the repository on GitHub:
```bash
# Go to https://github.com/juanjo-vlc and create a new repository named 'keycloak-argocd'
# Then push this repository:
git push -u origin main
```

### 2. Deploy with ArgoCD

Apply the ArgoCD Application manifest:
```bash
kubectl apply -f argocd-application.yaml --kubeconfig ~/.kube/anthrax.yaml
```

### 3. Verify Deployment

Check the ArgoCD Application status:
```bash
kubectl get application keycloak -n argocd --kubeconfig ~/.kube/anthrax.yaml
```

Check the PostgreSQL cluster:
```bash
kubectl get postgresql -n database --kubeconfig ~/.kube/anthrax.yaml
```

Check Keycloak pods:
```bash
kubectl get pods -n database -l app=keycloak --kubeconfig ~/.kube/anthrax.yaml
```

## Accessing Keycloak

Once deployed, Keycloak will be accessible at:
- URL: https://keycloak.garmo.local
- Admin username: admin
- Admin password: admin (change this immediately!)

## Database Credentials

The database credentials are managed by SOPS encryption. To view them:
```bash
sops -d keycloak-db-secret.yaml
```

## SOPS Configuration

The repository uses GPG key `41DB93BCEEA30326` for encryption. Make sure this key is:
1. Available in ArgoCD as a secret named `sops-gpg`
2. Imported in your local GPG keyring for encryption/decryption

## Architecture

### PostgreSQL HA
- 2 replicas with Zalando operator
- Automatic failover
- Database: keycloak
- User: keycloak (with superuser and createdb privileges)

### Keycloak HA
- 3 replicas using StatefulSet
- JGroups clustering for session replication
- Kubernetes DNS-based discovery
- Health checks enabled
- Metrics enabled

## Security Notes

- Change the default Keycloak admin password immediately after deployment
- Review and update resource limits based on your workload
- Ensure proper network policies are in place
- The database credentials are encrypted with SOPS but should be rotated regularly

## Troubleshooting

### PostgreSQL not starting
Check operator logs:
```bash
kubectl logs -n postgres-operator -l app.kubernetes.io/name=postgres-operator
```

### Keycloak pods not clustering
Check JGroups communication:
```bash
kubectl logs -n database keycloak-0 | grep jgroups
```

### SOPS decryption issues in ArgoCD
Verify the GPG key is correctly configured:
```bash
kubectl get secret sops-gpg -n argocd -o yaml
```
