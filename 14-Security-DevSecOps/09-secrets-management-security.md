# Secrets Management Security

## Secrets Management Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECRETS MANAGEMENT                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What are Secrets?                                                  │
│  • API keys and tokens                                             │
│  • Database credentials                                            │
│  • TLS certificates                                                │
│  • SSH keys                                                        │
│  • Encryption keys                                                 │
│  • Service account credentials                                     │
│                                                                     │
│  Secrets Lifecycle:                                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │Generate │─▶│ Store   │─▶│Distribute│─▶│  Use    │─▶│ Rotate  │ │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘ │
│       │                                                     │       │
│       └─────────────────────────────────────────────────────┘       │
│                         Continuous cycle                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## HashiCorp Vault

### Installation

```bash
# Docker
docker run -d --name vault \
  -p 8200:8200 \
  -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' \
  vault

# Kubernetes (Helm)
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault --set "server.dev.enabled=true"

# CLI
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='myroot'
```

### Basic Operations

```bash
# Enable secrets engine
vault secrets enable -path=secret kv-v2

# Store secret
vault kv put secret/myapp/config \
  username="admin" \
  password="supersecret"

# Read secret
vault kv get secret/myapp/config

# List secrets
vault kv list secret/myapp/

# Delete secret
vault kv delete secret/myapp/config
```

### Dynamic Secrets

```bash
# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@localhost:5432/mydb" \
  allowed_roles="readonly" \
  username="vault" \
  password="vaultpass"

# Create role
vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Generate credentials
vault read database/creds/readonly
```

### Kubernetes Integration

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: production
---
# Vault policy
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
---
# Kubernetes auth role
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp \
  bound_service_account_namespaces=production \
  policies=myapp-policy \
  ttl=1h
```

### Vault Agent Injector

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/myapp/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/myapp/config" -}}
          export DB_USER="{{ .Data.data.username }}"
          export DB_PASS="{{ .Data.data.password }}"
          {{- end }}
    spec:
      serviceAccountName: myapp
      containers:
        - name: app
          image: myapp:latest
          command: ["/bin/sh", "-c", "source /vault/secrets/config && ./start.sh"]
```

---

## AWS Secrets Manager

### CLI Usage

```bash
# Create secret
aws secretsmanager create-secret \
  --name myapp/database \
  --secret-string '{"username":"admin","password":"secret123"}'

# Get secret
aws secretsmanager get-secret-value \
  --secret-id myapp/database

# Rotate secret
aws secretsmanager rotate-secret \
  --secret-id myapp/database \
  --rotation-lambda-arn arn:aws:lambda:region:account:function:rotation

# Delete secret
aws secretsmanager delete-secret \
  --secret-id myapp/database \
  --force-delete-without-recovery
```

### Kubernetes External Secrets

```yaml
# External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secret
  data:
    - secretKey: username
      remoteRef:
        key: myapp/database
        property: username
    - secretKey: password
      remoteRef:
        key: myapp/database
        property: password
```

---

## Azure Key Vault

```bash
# Create Key Vault
az keyvault create \
  --name myapp-vault \
  --resource-group mygroup \
  --location eastus

# Store secret
az keyvault secret set \
  --vault-name myapp-vault \
  --name db-password \
  --value "supersecret"

# Get secret
az keyvault secret show \
  --vault-name myapp-vault \
  --name db-password
```

---

## Secret Rotation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECRET ROTATION                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Why Rotate Secrets?                                                │
│  • Limit exposure window if compromised                            │
│  • Compliance requirements                                         │
│  • Good security hygiene                                           │
│                                                                     │
│  Rotation Strategies:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Manual         │ Scheduled human intervention               │   │
│  │ Automated      │ Automatic rotation on schedule             │   │
│  │ Dynamic        │ Short-lived credentials (Vault)            │   │
│  │ On-demand      │ Rotate when needed                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Recommended Rotation Periods:                                      │
│  • API keys: 90 days                                               │
│  • Database passwords: 30-90 days                                  │
│  • TLS certificates: Before expiry (30 days)                       │
│  • Dynamic secrets: Hours (use short TTL)                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Zero-Trust Secrets

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ZERO-TRUST PRINCIPLES                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Never Trust, Always Verify                                      │
│     • Authenticate every secret request                            │
│     • Verify identity continuously                                 │
│                                                                     │
│  2. Least Privilege Access                                          │
│     • Grant minimum required permissions                           │
│     • Time-bound access                                            │
│     • Just-in-time provisioning                                    │
│                                                                     │
│  3. Assume Breach                                                   │
│     • Rotate secrets frequently                                    │
│     • Use short-lived credentials                                  │
│     • Audit all access                                             │
│                                                                     │
│  4. Encrypt Everything                                              │
│     • Encryption at rest                                           │
│     • Encryption in transit                                        │
│     • End-to-end encryption                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECRETS BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Storage:                                                           │
│  ✓ Use dedicated secrets management (Vault, AWS SM)                │
│  ✓ Encrypt secrets at rest                                         │
│  ✓ Never store in Git (even encrypted)                             │
│  ✓ Never log secrets                                               │
│                                                                     │
│  Access:                                                            │
│  ✓ Implement least privilege                                       │
│  ✓ Use identity-based access (not shared secrets)                  │
│  ✓ Audit all secret access                                         │
│  ✓ Time-bound access where possible                                │
│                                                                     │
│  Operations:                                                        │
│  ✓ Rotate secrets regularly                                        │
│  ✓ Use dynamic/short-lived secrets                                 │
│  ✓ Have incident response plan for leaks                           │
│  ✓ Test rotation procedures                                        │
│                                                                     │
│  Development:                                                       │
│  ✓ Use different secrets per environment                           │
│  ✓ Inject secrets at runtime                                       │
│  ✓ Never hardcode secrets in code                                  │
│  ✓ Use secret scanning in CI/CD                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Secrets Management Tools:                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ HashiCorp Vault │ Dynamic secrets, comprehensive            │   │
│  │ AWS Secrets Mgr │ AWS native, rotation support              │   │
│  │ Azure Key Vault │ Azure native, HSM backed                  │   │
│  │ GCP Secret Mgr  │ GCP native, IAM integration               │   │
│  │ External Secrets│ K8s operator for any backend              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Practices:                                                     │
│  • Use dedicated secrets management system                         │
│  • Implement rotation automation                                   │
│  • Prefer dynamic/short-lived secrets                              │
│  • Audit all access                                                │
│  • Never store secrets in Git                                      │
│                                                                     │
│  Next: Learn about infrastructure security scanning                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
