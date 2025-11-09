# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains Kubernetes manifests for deploying n8n (workflow automation platform) in a production environment. The deployment includes:
- n8n server with HTTPS/TLS support
- PostgreSQL database
- HashiCorp Vault integration for secrets management and PKI
- cert-manager for automated certificate management
- NFS-backed persistent storage

## Architecture

### Core Components

**n8n Server Deployment** (`deployment.n8n.server.yaml`)
- Runs n8n workflow automation platform
- Configured with HTTPS using local TLS certificates (no ingress)
- Uses PostgreSQL as the backend database
- Persistent storage via NFS-backed PVC (`n8n-claim0`)
- Init container handles volume permissions (fsGroup 3000)
- Health checks on `/healthz` endpoint
- Metrics enabled via `N8N_METRICS=true`
- Git version control integration for workflow versioning (requires SSH key in `n8n-secret`)

**PostgreSQL Deployment** (`deployment.n8n.postgres.yaml`)
- PostgreSQL 11 database for n8n data persistence
- NFS-backed persistent volume (`postgresql-pv`)
- Custom init script via ConfigMap to create non-root user
- Separate root and non-root credentials via secrets

### Security & Secrets

**Vault Integration**
- Uses ArgoCD Vault Plugin (AVP) for secret injection
- Annotation: `avp.kubernetes.io/path: "argocd/data/argocd-vault-plugin/n8n"`
- Secrets are stored in Vault and injected at deployment time
- Database credentials managed through `postgres-secret`

**TLS/Certificate Management**
- cert-manager handles certificate lifecycle
- Vault PKI backend issues certificates via `vault-issuer` (Issuer in n8n namespace)
- Certificates stored in `n8n-tls-secret` in n8n namespace
- n8n serves HTTPS directly (no ingress controller)
- Service account `cert-manager` in n8n namespace authenticates to Vault
- Certificate uses wildcard DNS pattern (configured in Vault placeholders)

**Required Vault Configuration**
The cert-manager service account requires proper Vault authentication setup:
```bash
# Create Vault policy for cert-manager
vault policy write cert-manager-policy - <<EOF
path "pki_int/sign/cert-manager" {
  capabilities = ["create", "update"]
}
path "pki_int/issue/cert-manager" {
  capabilities = ["create", "update"]
}
EOF

# Create Kubernetes auth role
vault write auth/kubernetes/role/cert-manager \
  bound_service_account_names=cert-manager \
  bound_service_account_namespaces=n8n \
  policies=cert-manager-policy \
  ttl=24h
```

### Storage

**Persistent Volumes**
- `truenas-n8n-pv`: NFS volume for n8n data (50Gi, ReadWriteMany)
- `postgresql-pv`: NFS volume for PostgreSQL data
- Both use TrueNAS NFS backend (configured via Vault placeholders)

**ConfigMap**
- `init-data` (cm.n8n.postgres.yaml): PostgreSQL initialization script
- Creates non-root user with appropriate database privileges

## Common Operations

### Deploying the Stack

Apply resources in this order:
```bash
# 1. Namespace and RBAC
kubectl apply -f ns.n8n.yaml
kubectl apply -f sa.n8n.yaml

# 2. Storage
kubectl apply -f pv.n8n.yaml
kubectl apply -f pvc.n8n.postgres.yaml
kubectl apply -f pvc.n8n.dev.yaml

# 3. Secrets and ConfigMaps
kubectl apply -f secret.n8n.postgres.yaml
kubectl apply -f cm.n8n.postgres.yaml

# 4. Certificate management
kubectl apply -f issuer.n8n.yaml
kubectl apply -f certificate.n8n.yaml

# 5. Services
kubectl apply -f service.n8n.postgres.yaml
kubectl apply -f service.n8n.server.yaml

# 6. Deployments
kubectl apply -f deployment.n8n.postgres.yaml
kubectl apply -f deployment.n8n.server.yaml
```

### Viewing Resources

```bash
# Check all resources in n8n namespace
kubectl get all -n n8n

# Check certificates
kubectl get certificate -n n8n
kubectl describe certificate n8n-tls -n n8n

# Check issuer status
kubectl get issuer -n n8n
kubectl describe issuer vault-issuer -n n8n

# Check secrets (TLS and database)
kubectl get secrets -n n8n

# Check persistent volumes
kubectl get pv | grep n8n
kubectl get pvc -n n8n
```

### Debugging

```bash
# Check n8n logs
kubectl logs -n n8n deployment/n8n -f

# Check PostgreSQL logs
kubectl logs -n n8n deployment/postgres -f

# Check certificate issuance
kubectl describe certificate n8n-tls -n n8n
kubectl get certificaterequest -n n8n

# Debug Vault issuer (403 permission denied errors)
kubectl describe issuer vault-issuer -n n8n
# Check Vault logs for authentication issues
# Verify Vault role exists: vault list auth/kubernetes/role
# Verify service account can authenticate: vault read auth/kubernetes/role/cert-manager

# Check Git version control integration
kubectl exec -n n8n deployment/n8n -- sh -c 'ls -la /home/node/.n8n/.git'
kubectl logs -n n8n deployment/n8n | grep -i git

# Verify TLS certificates are mounted correctly
kubectl exec -n n8n deployment/n8n -- ls -la /etc/ssl/certs/n8n/
```

### Updating Configuration

When modifying environment variables or secrets:
```bash
# Edit deployment
kubectl edit deployment n8n -n n8n

# Or apply updated manifest
kubectl apply -f deployment.n8n.server.yaml

# Force rollout restart if secrets changed
kubectl rollout restart deployment/n8n -n n8n
kubectl rollout restart deployment/postgres -n n8n
```

## Configuration Placeholders

Several manifests contain placeholders that must be replaced before deployment:
- `<n8n_internal_url>`: Internal URL for n8n server
- `<n8n_internal_postgres_url>`: Internal PostgreSQL service URL (typically `postgres-service`)
- `<vault_url>`: HashiCorp Vault server URL
- `<pki_sign_int_path>`: Vault PKI signing path (e.g., `pki_int/sign/cert-manager`)
- `<git_user_name>`: Git username for version control commits
- `<git_user_email>`: Git email for version control commits
- `<git_repo_url>`: SSH Git repository URL for workflow version control
- `<argocd/data/argocd-vault-plugin/network/wildcard_domain>`: Wildcard domain for certificates

These are typically injected by ArgoCD Vault Plugin from Vault secrets.

## Git Version Control Integration

n8n supports built-in Git version control for workflow versioning:

**Configuration** (deployment.n8n.server.yaml:41-53)
- Enabled via `N8N_VERSION_CONTROL_ENABLED=true`
- Requires SSH private key stored in `n8n-secret` with key `N8N_VERSION_CONTROL_GIT_PRIVATE_KEY`
- Git credentials configured via environment variables (user name, email, repo URL)

**SSH Key Management**
- Generate SSH key pair: `ssh-keygen -t ed25519 -f n8n_id_ed25519 -C "n8n-version-control"`
- Add public key to Git repository (GitHub/GitLab deploy keys)
- Store private key in Vault, injected into `n8n-secret` via AVP
- n8n will automatically commit workflow changes to the configured repository

**Important**: SSH keys in this directory (n8n_id_ed25519, n8n_id_ed25519.pub) should NOT be committed to version control. They are deployment artifacts. Add them to .gitignore:
```
# .gitignore
n8n_id_ed25519
n8n_id_ed25519.pub
```

## Important Notes

- n8n uses `Recreate` deployment strategy to avoid multiple instances writing to same volume
- PostgreSQL init script runs only on first container start
- Volume permissions are managed by init container (chown 3000:3000)
- n8n runs as non-root user (UID/GID 3000)
- Data retention: Execution data pruned after 168 hours (7 days)
- Resource limits: n8n can use up to 4 CPU cores and 4Gi memory
