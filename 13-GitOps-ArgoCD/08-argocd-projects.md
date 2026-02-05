# ArgoCD Projects

## What is an AppProject?

An AppProject is a logical grouping of applications that provides RBAC, source/destination restrictions, and resource whitelisting.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    APPPROJECT CONCEPT                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  AppProject provides:                                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  RBAC                    │ Who can do what                  │   │
│  │  Source Restrictions     │ Which Git repos are allowed      │   │
│  │  Destination Restrictions│ Which clusters/namespaces        │   │
│  │  Resource Whitelist      │ Which K8s resources can deploy   │   │
│  │  Sync Windows            │ When sync is allowed             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Multi-tenancy Model:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      ArgoCD Instance                        │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │   │
│  │  │  Project A   │ │  Project B   │ │  Project C   │        │   │
│  │  │  Team Alpha  │ │  Team Beta   │ │  Team Gamma  │        │   │
│  │  │  ──────────  │ │  ──────────  │ │  ──────────  │        │   │
│  │  │  app-1       │ │  app-3       │ │  app-5       │        │   │
│  │  │  app-2       │ │  app-4       │ │  app-6       │        │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Default Project

Every ArgoCD installation includes a `default` project with no restrictions.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
  namespace: argocd
spec:
  description: Default project (no restrictions)
  sourceRepos:
    - '*'  # All repositories allowed
  destinations:
    - namespace: '*'
      server: '*'  # All clusters and namespaces allowed
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'  # All cluster resources allowed
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEFAULT PROJECT                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  WARNING: The default project has no restrictions!                  │
│                                                                     │
│  Best Practices:                                                    │
│  • Don't use default project for production applications           │
│  • Create specific projects with appropriate restrictions          │
│  • Lock down default project or remove it entirely                 │
│                                                                     │
│  To lock down default project:                                      │
│  sourceRepos: []      # No repos allowed                           │
│  destinations: []     # No destinations allowed                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Creating Projects

### Basic Project

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production applications

  # Allowed source repositories
  sourceRepos:
    - https://github.com/myorg/production-apps.git
    - https://github.com/myorg/shared-configs.git

  # Allowed destinations
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
    - namespace: production-*
      server: https://kubernetes.default.svc
```

### Complete Project Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-frontend
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Frontend team applications

  # Source repositories
  sourceRepos:
    - https://github.com/myorg/frontend-*
    - https://charts.helm.sh/stable

  # Destination clusters and namespaces
  destinations:
    - namespace: frontend-*
      server: https://kubernetes.default.svc
    - namespace: frontend-prod
      server: https://prod-cluster.example.com

  # Cluster-scoped resources allowed
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding

  # Namespace-scoped resources denied
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
    - group: ''
      kind: LimitRange

  # Roles for this project
  roles:
    - name: developer
      description: Developer access
      policies:
        - p, proj:team-frontend:developer, applications, get, team-frontend/*, allow
        - p, proj:team-frontend:developer, applications, sync, team-frontend/*, allow
      groups:
        - frontend-developers

    - name: admin
      description: Admin access
      policies:
        - p, proj:team-frontend:admin, applications, *, team-frontend/*, allow
      groups:
        - frontend-admins

  # Sync windows
  syncWindows:
    - kind: allow
      schedule: '0 9-17 * * 1-5'
      duration: 8h
      applications:
        - '*'
```

### Create via CLI

```bash
# Create basic project
argocd proj create production \
  --description "Production applications" \
  --src https://github.com/myorg/apps.git \
  --dest https://kubernetes.default.svc,production

# Add more sources
argocd proj add-source production https://github.com/myorg/configs.git

# Add more destinations
argocd proj add-destination production https://kubernetes.default.svc,production-db

# Allow cluster resources
argocd proj allow-cluster-resource production "" Namespace
argocd proj allow-cluster-resource production rbac.authorization.k8s.io ClusterRole
```

---

## Source Repository Restrictions

### Patterns for sourceRepos

```yaml
spec:
  sourceRepos:
    # Exact match
    - https://github.com/myorg/my-repo.git

    # Wildcard - all repos in org
    - https://github.com/myorg/*

    # Wildcard - all GitHub repos
    - https://github.com/*/*

    # All repos (not recommended)
    - '*'

    # Helm repositories
    - https://charts.helm.sh/stable
    - https://prometheus-community.github.io/helm-charts

    # OCI registries
    - oci://ghcr.io/myorg/*
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SOURCE REPO PATTERNS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pattern Examples:                                                  │
│                                                                     │
│  https://github.com/org/repo.git    → Exact repo                   │
│  https://github.com/org/*           → All repos in org             │
│  https://github.com/org/prefix-*    → Repos starting with prefix   │
│  https://github.com/*/*             → All GitHub repos             │
│  *                                  → Any repository               │
│                                                                     │
│  Best Practice: Be as specific as possible                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Destination Restrictions

### Destination Configuration

```yaml
spec:
  destinations:
    # Specific namespace on default cluster
    - namespace: production
      server: https://kubernetes.default.svc

    # Wildcard namespace
    - namespace: team-a-*
      server: https://kubernetes.default.svc

    # All namespaces on specific cluster
    - namespace: '*'
      server: https://prod-cluster.example.com

    # Using cluster name instead of URL
    - namespace: production
      name: production-cluster

    # Deny all (empty list)
    # destinations: []
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DESTINATION PATTERNS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  namespace: production              → Exact namespace              │
│  namespace: team-*                  → Namespaces starting with     │
│  namespace: '*'                     → All namespaces               │
│  namespace: '!kube-system'          → NOT kube-system (negation)   │
│                                                                     │
│  server: https://kubernetes.default.svc  → In-cluster             │
│  server: https://prod.example.com        → External cluster        │
│  server: '*'                             → All clusters            │
│  name: my-cluster                        → By cluster name         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Resource Restrictions

### Cluster Resource Whitelist

```yaml
spec:
  # Cluster-scoped resources that can be created
  clusterResourceWhitelist:
    # Allow Namespaces
    - group: ''
      kind: Namespace

    # Allow RBAC
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding

    # Allow CRDs
    - group: apiextensions.k8s.io
      kind: CustomResourceDefinition

    # Allow all cluster resources (not recommended)
    # - group: '*'
    #   kind: '*'
```

### Cluster Resource Blacklist

```yaml
spec:
  # Cluster-scoped resources that are denied
  clusterResourceBlacklist:
    - group: ''
      kind: Namespace  # Prevent namespace creation
    - group: rbac.authorization.k8s.io
      kind: '*'  # Prevent all RBAC changes
```

### Namespace Resource Whitelist/Blacklist

```yaml
spec:
  # Namespace-scoped resources allowed (whitelist takes precedence)
  namespaceResourceWhitelist:
    - group: ''
      kind: ConfigMap
    - group: ''
      kind: Secret
    - group: apps
      kind: Deployment
    - group: ''
      kind: Service

  # Namespace-scoped resources denied
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
    - group: ''
      kind: LimitRange
    - group: ''
      kind: PersistentVolumeClaim
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RESOURCE RESTRICTION LOGIC                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Cluster Resources:                                                 │
│  • If clusterResourceWhitelist is set → only listed allowed        │
│  • If clusterResourceBlacklist is set → listed are denied          │
│  • Empty whitelist = no cluster resources allowed                  │
│                                                                     │
│  Namespace Resources:                                               │
│  • By default, all namespace resources allowed                     │
│  • namespaceResourceBlacklist → deny specific resources            │
│  • namespaceResourceWhitelist → only allow specific resources      │
│  • Whitelist takes precedence over blacklist                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Project Roles and RBAC

### Defining Roles

```yaml
spec:
  roles:
    # Read-only role
    - name: viewer
      description: Read-only access to applications
      policies:
        - p, proj:myproject:viewer, applications, get, myproject/*, allow
      groups:
        - my-team-viewers

    # Developer role
    - name: developer
      description: Can sync applications
      policies:
        - p, proj:myproject:developer, applications, get, myproject/*, allow
        - p, proj:myproject:developer, applications, sync, myproject/*, allow
        - p, proj:myproject:developer, applications, action/*, myproject/*, allow
        - p, proj:myproject:developer, logs, get, myproject/*, allow
      groups:
        - my-team-developers

    # Admin role
    - name: admin
      description: Full access to applications
      policies:
        - p, proj:myproject:admin, applications, *, myproject/*, allow
        - p, proj:myproject:admin, logs, get, myproject/*, allow
        - p, proj:myproject:admin, exec, create, myproject/*, allow
      groups:
        - my-team-admins
```

### Policy Syntax

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RBAC POLICY SYNTAX                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  p, <role>, <resource>, <action>, <appproject>/<object>, <effect>  │
│                                                                     │
│  Role:      proj:<project>:<role-name>                             │
│  Resource:  applications, logs, exec, repositories, clusters       │
│  Action:    get, create, update, delete, sync, action/*, *         │
│  Object:    application name or * for all                          │
│  Effect:    allow or deny                                          │
│                                                                     │
│  Examples:                                                          │
│  ───────────────────────────────────────────────────────────────   │
│  # View all apps in project                                        │
│  p, proj:prod:viewer, applications, get, prod/*, allow             │
│                                                                     │
│  # Sync specific app                                                │
│  p, proj:prod:dev, applications, sync, prod/frontend, allow        │
│                                                                     │
│  # Full access to all apps in project                              │
│  p, proj:prod:admin, applications, *, prod/*, allow                │
│                                                                     │
│  # View logs                                                        │
│  p, proj:prod:dev, logs, get, prod/*, allow                        │
│                                                                     │
│  # Execute commands in pods                                         │
│  p, proj:prod:admin, exec, create, prod/*, allow                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Group Bindings

```yaml
spec:
  roles:
    - name: developer
      policies:
        - p, proj:myproject:developer, applications, *, myproject/*, allow
      # SSO groups that get this role
      groups:
        - github:myorg/developers    # GitHub team
        - gitlab:mygroup/developers  # GitLab group
        - okta:developers            # Okta group
        - ldap:cn=developers,ou=groups,dc=example,dc=com  # LDAP
```

### JWT Token for CI/CD

```yaml
spec:
  roles:
    - name: ci-role
      description: CI/CD automation role
      policies:
        - p, proj:myproject:ci-role, applications, sync, myproject/*, allow
        - p, proj:myproject:ci-role, applications, get, myproject/*, allow
      # Generate JWT tokens for this role
      jwtTokens:
        - iat: 1234567890  # Issued at timestamp
          id: ci-token
```

```bash
# Generate token via CLI
argocd proj role create-token myproject ci-role

# Use token in CI/CD
export ARGOCD_AUTH_TOKEN="<generated-token>"
argocd app sync myapp --auth-token $ARGOCD_AUTH_TOKEN
```

---

## Global RBAC Configuration

### argocd-rbac-cm ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  # Default policy for authenticated users
  policy.default: role:readonly

  # Policy definitions
  policy.csv: |
    # Admin group gets admin role
    g, admins, role:admin

    # Developers can sync
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    g, developers, role:developer

    # Project-specific roles
    p, role:team-a-admin, applications, *, team-a/*, allow
    g, team-a-admins, role:team-a-admin

    # Deny dangerous operations
    p, role:developer, clusters, *, *, deny
    p, role:developer, repositories, delete, *, deny

  # Scopes to check for group membership
  scopes: '[groups, email]'
```

### Built-in Roles

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BUILT-IN ROLES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  role:readonly                                                      │
│  ─────────────                                                      │
│  • View applications, projects, repositories, clusters             │
│  • Cannot modify anything                                          │
│                                                                     │
│  role:admin                                                         │
│  ──────────                                                         │
│  • Full access to everything                                       │
│  • Create, update, delete all resources                            │
│  • Manage settings                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Project Patterns

### Environment-Based Projects

```yaml
# Development project
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: development
spec:
  sourceRepos:
    - '*'
  destinations:
    - namespace: '*'
      server: https://dev-cluster.example.com
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
---
# Production project (restricted)
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
spec:
  sourceRepos:
    - https://github.com/myorg/production-configs.git
  destinations:
    - namespace: production-*
      server: https://prod-cluster.example.com
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  syncWindows:
    - kind: deny
      schedule: '* * * * *'
      duration: 24h
      manualSync: true
```

### Team-Based Projects

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-payments
spec:
  description: Payments team applications
  sourceRepos:
    - https://github.com/myorg/payments-*
  destinations:
    - namespace: payments-*
      server: '*'
  roles:
    - name: team-member
      policies:
        - p, proj:team-payments:team-member, applications, *, team-payments/*, allow
      groups:
        - payments-team
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  AppProject Features:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ sourceRepos              │ Allowed Git repositories         │   │
│  │ destinations             │ Allowed clusters/namespaces      │   │
│  │ clusterResourceWhitelist │ Allowed cluster-scoped resources │   │
│  │ namespaceResourceBlacklist│ Denied namespace resources      │   │
│  │ roles                    │ Project-specific RBAC            │   │
│  │ syncWindows              │ Sync time restrictions           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Best Practices:                                                    │
│  • Don't use default project for real applications                 │
│  • Create project per team or environment                          │
│  • Be specific with source and destination restrictions            │
│  • Use roles for fine-grained access control                       │
│  • Integrate with SSO groups                                       │
│                                                                     │
│  Next: Learn about repository management                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
