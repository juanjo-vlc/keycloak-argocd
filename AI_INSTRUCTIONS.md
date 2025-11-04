# AI Agent Instructions for Keycloak ArgoCD Deployment

This document provides guidelines for AI agents working with this Kubernetes infrastructure setup.

## Overview

This repository manages a High Availability Keycloak deployment with:
- PostgreSQL database with Zalando operator
- PgBouncer connection pooling
- SOPS-encrypted secrets
- GitOps deployment via ArgoCD

## Deployment Strategy

### 1. Use ArgoCD Applications for All Deployments

**Rule**: Never apply manifests directly with `kubectl apply`. Always use ArgoCD Applications.

#### Structure
- All deployments MUST be managed through ArgoCD Applications
- Repository is organized by namespace:
  - `database/` - PostgreSQL cluster and database resources
  - `keycloak/` - Keycloak application resources
- Each namespace has its own ArgoCD Application in [argocd-application.yaml](argocd-application.yaml)

#### Creating New Applications
When adding new applications:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/juanjo-vlc/keycloak-argocd.git
    targetRevision: HEAD
    path: <namespace-directory>
  destination:
    server: https://kubernetes.default.svc
    namespace: <target-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

#### Deployment Workflow
1. Make changes to manifests in the repository
2. Commit and push to GitHub
3. Sync ArgoCD application:
   ```bash
   kubectl -n argocd patch application <app-name> --type merge \
     -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
   ```

## Secret Management with SOPS

### 2. Always Use SOPS for Secrets

**Rule**: All Kubernetes secrets MUST be encrypted with SOPS using the configured GPG key.

#### Current Configuration
- **GPG Key Fingerprint**: `41DB93BCEEA30326`
- **SOPS Config**: [.sops.yaml](.sops.yaml)
- **Encrypted Regex**: `^(data|stringData)$`

#### SOPS Workflow

**Creating/Updating Secrets**:

1. Create plain secret file:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: my-namespace
type: Opaque
stringData:
  key1: value1
  key2: value2
```

2. Encrypt with SOPS:
```bash
sops -e plain-secret.yaml > encrypted-secret.yaml
```

3. To update existing secret:
```bash
# Decrypt
sops -d encrypted-secret.yaml > plain-secret.yaml

# Edit plain-secret.yaml

# Re-encrypt
sops -e plain-secret.yaml > encrypted-secret.yaml
```

#### Kustomize Integration with KSOPS

**Important**: This repository uses KSOPS plugin for ArgoCD to decrypt secrets automatically.

Each namespace directory MUST have:

1. **kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - other-resources.yaml

generators:
  - secret-generator.yaml
```

2. **secret-generator.yaml**:
```yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: secret-generator
  annotations:
    config.kubernetes.io/function: |
      exec:
        path: ksops
files:
  - my-encrypted-secret.yaml
```

3. **my-encrypted-secret.yaml**: Your SOPS-encrypted secret file

**Never** list encrypted secrets directly in `resources:` - they MUST go through the `generators:` section.

## Database Configuration

### 3. Always Use Connection Pooler for Database Access

**Rule**: When configuring database access, ALWAYS use PgBouncer connection pooler endpoints, never direct PostgreSQL services.

#### Current Setup
- PostgreSQL Cluster: `keycloak-postgres` (2 replicas)
- PgBouncer Poolers: 2 primary + 2 replica instances
- Pooling Mode: **session** (required for prepared statements)

#### Connection Endpoints

**Use these services for application connections**:
- **Primary (read-write)**: `keycloak-postgres-pooler.database.svc.cluster.local:5432`
- **Replica (read-only)**: `keycloak-postgres-pooler-repl.database.svc.cluster.local:5432`

**DO NOT use these** (direct PostgreSQL endpoints):
- ❌ `keycloak-postgres.database.svc.cluster.local`
- ❌ `keycloak-postgres-repl.database.svc.cluster.local`

#### Database Secret Format

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-db-credentials
  namespace: <app-namespace>
type: Opaque
stringData:
  DB_VENDOR: postgres
  DB_ADDR: keycloak-postgres-pooler.database.svc.cluster.local
  DB_PORT: "5432"
  DB_DATABASE: <database-name>
  DB_USER: <username>
  DB_PASSWORD: <password>
  JDBC_URL: jdbc:postgresql://keycloak-postgres-pooler.database.svc.cluster.local:5432/<database-name>
```

#### PgBouncer Configuration

When modifying PostgreSQL cluster configuration in [database/postgres-cluster.yaml](database/postgres-cluster.yaml):

```yaml
spec:
  enableConnectionPooler: true
  enableReplicaConnectionPooler: true
  connectionPooler:
    numberOfInstances: 2
    mode: "session"  # REQUIRED - do not change to "transaction"
    schema: "pooler"
    user: "pooler"
    maxDBConnections: 60
    resources:
      requests:
        cpu: 100m
        memory: 100Mi
      limits:
        cpu: 200m
        memory: 200Mi
```

**Important**:
- Mode MUST be `session` for applications using prepared statements (like Keycloak, Liquibase)
- Transaction mode does NOT support prepared statements
- Do not disable connection pooling without explicit approval

## Common Operations

### Adding a New Application

1. Create namespace directory: `mkdir -p <namespace>/`
2. Create manifest files in the directory
3. Create `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml

generators:
  - secret-generator.yaml  # if you have secrets
```

4. If using secrets:
   - Create plain secret YAML
   - Encrypt with SOPS
   - Create `secret-generator.yaml` referencing the encrypted file

5. Add ArgoCD Application to [argocd-application.yaml](argocd-application.yaml)

6. Commit and push:
```bash
git add .
git commit -m "Add <app-name> application"
git push origin main
```

7. Apply ArgoCD application:
```bash
kubectl apply -f argocd-application.yaml
```

### Updating Database Credentials

1. Decrypt existing secret:
```bash
sops -d database/keycloak-db-secret.yaml > /tmp/secret.yaml
```

2. Update values in `/tmp/secret.yaml`

3. Re-encrypt:
```bash
sops -e /tmp/secret.yaml > database/keycloak-db-secret.yaml
```

4. Also update keycloak namespace copy:
```bash
sops -e /tmp/secret.yaml > keycloak/keycloak-db-secret.yaml
```

5. Commit, push, and sync:
```bash
git add database/keycloak-db-secret.yaml keycloak/keycloak-db-secret.yaml
git commit -m "Update database credentials"
git push origin main
kubectl -n argocd patch application keycloak-database --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
kubectl -n argocd patch application keycloak --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
```

### Troubleshooting

#### SOPS Decryption Issues
If secrets are not being decrypted (you see `ENC[...]` values in pods):
1. Verify KSOPS is installed in ArgoCD repo-server:
   ```bash
   kubectl exec -n argocd argocd-repo-server-<pod> -- which sops
   ```
2. Check kustomization.yaml uses `generators:` not `resources:` for secrets
3. Verify secret-generator.yaml exists and references the encrypted file

#### PgBouncer Connection Issues
If applications can't connect to database:
1. Check PgBouncer pods are running:
   ```bash
   kubectl get pods -n database -l connection-pooler=keycloak-postgres-pooler
   ```
2. Verify connection string uses pooler service:
   ```bash
   kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.DB_ADDR}' | base64 -d
   ```
   Should output: `keycloak-postgres-pooler.database.svc.cluster.local`

3. Check PgBouncer logs:
   ```bash
   kubectl logs -n database -l connection-pooler=keycloak-postgres-pooler
   ```

#### Prepared Statement Errors
If you see errors like "prepared statement already exists":
- PgBouncer is in transaction mode (incorrect)
- Change to session mode in [database/postgres-cluster.yaml](database/postgres-cluster.yaml)
- Set `connectionPooler.mode: "session"`

## Security Best Practices

1. **Never commit unencrypted secrets** to the repository
2. **Always use SOPS** for any sensitive data
3. **Rotate credentials regularly** by updating encrypted secrets
4. **Use GPG key** `41DB93BCEEA30326` for all SOPS operations
5. **Test encryption** before committing: `sops -d <file>` should work without errors

## GitOps Workflow Summary

```
┌─────────────────┐
│  Make Changes   │
│  to Manifests   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Encrypt Secrets │
│   with SOPS     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Commit & Push   │
│   to GitHub     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Sync ArgoCD    │
│  Application    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Verify in     │
│    Cluster      │
└─────────────────┘
```

## Quick Reference

### Key Files
- [argocd-application.yaml](argocd-application.yaml) - ArgoCD applications
- [.sops.yaml](.sops.yaml) - SOPS configuration
- [database/postgres-cluster.yaml](database/postgres-cluster.yaml) - PostgreSQL & PgBouncer config
- [database/kustomization.yaml](database/kustomization.yaml) - Database namespace resources
- [keycloak/kustomization.yaml](keycloak/kustomization.yaml) - Keycloak namespace resources

### Key Services
- PgBouncer Primary: `keycloak-postgres-pooler.database.svc.cluster.local:5432`
- PgBouncer Replica: `keycloak-postgres-pooler-repl.database.svc.cluster.local:5432`
- Keycloak: `keycloak.keycloak.svc.cluster.local:8080`
- Ingress: `https://keycloak.garmo.local`

### Commands
```bash
# Encrypt secret
sops -e plain.yaml > encrypted.yaml

# Decrypt secret
sops -d encrypted.yaml > plain.yaml

# Sync ArgoCD app
kubectl -n argocd patch application <name> --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# Check ArgoCD app status
kubectl get application -n argocd

# Check PgBouncer
kubectl get pods -n database -l connection-pooler=keycloak-postgres-pooler
```

## Questions or Issues?

Refer to the main [README.md](README.md) for detailed setup instructions and troubleshooting steps.
