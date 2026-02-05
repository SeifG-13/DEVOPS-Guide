# GitOps Fundamentals

## What is GitOps?

GitOps is a paradigm for managing infrastructure and application deployments using Git as the single source of truth.

```
┌─────────────────────────────────────────────────────────────┐
│                      GITOPS DEFINED                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  "GitOps is a way of implementing Continuous Deployment     │
│   for cloud-native applications. It focuses on a           │
│   developer-centric experience when operating              │
│   infrastructure, by using tools developers are already    │
│   familiar with, including Git and CI/CD tools."           │
│                                                             │
│                              - Weaveworks (coined in 2017)  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Core Principles of GitOps

### The Four Principles

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FOUR PRINCIPLES OF GITOPS                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. DECLARATIVE                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ The entire system must be described declaratively           │   │
│  │ • Infrastructure as Code (IaC)                              │   │
│  │ • Kubernetes manifests, Helm charts, Kustomize             │   │
│  │ • "What" not "How"                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  2. VERSIONED AND IMMUTABLE                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Git as the single source of truth                           │   │
│  │ • Complete version history                                  │   │
│  │ • Immutable configuration                                   │   │
│  │ • Easy rollback to any previous state                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  3. PULLED AUTOMATICALLY                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Approved changes are automatically applied to the system    │   │
│  │ • Software agents pull changes                              │   │
│  │ • No human intervention needed                              │   │
│  │ • Continuous reconciliation                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  4. CONTINUOUSLY RECONCILED                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Software agents ensure actual state matches desired state   │   │
│  │ • Self-healing systems                                      │   │
│  │ • Drift detection and correction                            │   │
│  │ • Continuous monitoring                                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Git as Single Source of Truth

### Why Git?

```
┌─────────────────────────────────────────────────────────────┐
│                  GIT AS SOURCE OF TRUTH                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    GIT REPOSITORY                    │   │
│  │                                                      │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │   │
│  │  │  Version   │  │   Audit    │  │  Branch    │    │   │
│  │  │  Control   │  │   Trail    │  │  Strategy  │    │   │
│  │  └────────────┘  └────────────┘  └────────────┘    │   │
│  │                                                      │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │   │
│  │  │   Pull     │  │  Access    │  │ Rollback   │    │   │
│  │  │  Requests  │  │  Control   │  │  Ability   │    │   │
│  │  └────────────┘  └────────────┘  └────────────┘    │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Benefits:                                                  │
│  • Complete history of all changes                         │
│  • Who changed what, when, and why                         │
│  • Code review process for infrastructure                  │
│  • Easy collaboration with familiar tools                  │
│  • Built-in security and access control                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Repository Structure Example

```
gitops-repo/
├── apps/
│   ├── production/
│   │   ├── frontend/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── ingress.yaml
│   │   └── backend/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   ├── staging/
│   │   └── ...
│   └── development/
│       └── ...
├── infrastructure/
│   ├── namespaces/
│   ├── rbac/
│   └── monitoring/
└── base/
    ├── frontend/
    └── backend/
```

---

## Push vs Pull Model

### Traditional Push-Based CI/CD

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PUSH-BASED MODEL (Traditional)                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Developer          CI System              Cluster                  │
│                                                                     │
│  ┌────────┐        ┌────────┐            ┌────────┐                │
│  │  Code  │──────▶ │  Build │──────────▶ │  K8s   │                │
│  │ Change │  push  │  Test  │   push     │Cluster │                │
│  └────────┘        │ Deploy │  (kubectl) └────────┘                │
│                    └────────┘                                       │
│                                                                     │
│  Problems:                                                          │
│  • CI system needs cluster credentials                             │
│  • Security risk: credentials outside cluster                      │
│  • No drift detection                                              │
│  • No self-healing                                                 │
│  • Cluster state can diverge from Git                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### GitOps Pull-Based Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PULL-BASED MODEL (GitOps)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Developer          Git Repo              Cluster                   │
│                                                                     │
│  ┌────────┐        ┌────────┐            ┌────────────────────┐    │
│  │  Code  │──────▶ │  Git   │◀─────────  │    Kubernetes      │    │
│  │ Change │  push  │  Repo  │   pull     │  ┌──────────────┐  │    │
│  └────────┘        └────────┘            │  │ GitOps Agent │  │    │
│                                          │  │  (ArgoCD/    │  │    │
│                                          │  │   Flux)      │  │    │
│                                          │  └──────────────┘  │    │
│                                          └────────────────────┘    │
│                                                                     │
│  Benefits:                                                          │
│  • Credentials stay inside cluster                                 │
│  • Improved security posture                                       │
│  • Continuous drift detection                                      │
│  • Self-healing capability                                         │
│  • Git always reflects cluster state                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Comparison Table

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PUSH vs PULL COMPARISON                           │
├───────────────────┬─────────────────────┬───────────────────────────┤
│ Aspect            │ Push-Based          │ Pull-Based (GitOps)       │
├───────────────────┼─────────────────────┼───────────────────────────┤
│ Deployment        │ CI pushes to        │ Agent pulls from Git      │
│ Trigger           │ cluster             │                           │
├───────────────────┼─────────────────────┼───────────────────────────┤
│ Credentials       │ Stored in CI        │ Stored in cluster         │
│ Location          │ system              │                           │
├───────────────────┼─────────────────────┼───────────────────────────┤
│ Drift             │ Manual detection    │ Automatic detection       │
│ Detection         │                     │ and correction            │
├───────────────────┼─────────────────────┼───────────────────────────┤
│ Security          │ Broader attack      │ Reduced attack            │
│                   │ surface             │ surface                   │
├───────────────────┼─────────────────────┼───────────────────────────┤
│ Recovery          │ Re-run pipeline     │ Automatic                 │
│                   │                     │ reconciliation            │
├───────────────────┼─────────────────────┼───────────────────────────┤
│ Audit             │ Scattered across    │ Complete in Git           │
│ Trail             │ CI logs             │ history                   │
└───────────────────┴─────────────────────┴───────────────────────────┘
```

---

## GitOps vs Traditional CI/CD

### Traditional CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TRADITIONAL CI/CD                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐            │
│  │ Code │──▶│Build │──▶│ Test │──▶│Build │──▶│Deploy│            │
│  │Commit│   │      │   │      │   │Image │   │      │            │
│  └──────┘   └──────┘   └──────┘   └──────┘   └──────┘            │
│                                                   │                 │
│                                                   ▼                 │
│                                            ┌───────────┐           │
│                                            │  Cluster  │           │
│                                            └───────────┘           │
│                                                                     │
│  Pipeline does: Build → Test → Deploy (all in one)                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### GitOps Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GITOPS PIPELINE                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CI Pipeline (Continuous Integration)                               │
│  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌────────────┐       │
│  │ Code │──▶│Build │──▶│ Test │──▶│Build │──▶│ Push Image │       │
│  │Commit│   │      │   │      │   │Image │   │ to Registry│       │
│  └──────┘   └──────┘   └──────┘   └──────┘   └─────┬──────┘       │
│                                                     │               │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─  │
│                                                     ▼               │
│  CD Pipeline (Continuous Delivery/Deployment)       │               │
│                                               ┌─────┴─────┐         │
│                                               │Update Git │         │
│                                               │  Manifests│         │
│                                               └─────┬─────┘         │
│                                                     │               │
│                                                     ▼               │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                     CONFIG REPOSITORY                         │ │
│  │                                                               │ │
│  │  deployment.yaml: image: myapp:v2.0.0                        │ │
│  │                                                               │ │
│  └────────────────────────────┬──────────────────────────────────┘ │
│                               │                                     │
│                    ┌──────────▼──────────┐                         │
│                    │    GitOps Agent     │                         │
│                    │   (ArgoCD/Flux)     │                         │
│                    │                     │                         │
│                    │  Detects change,    │                         │
│                    │  syncs to cluster   │                         │
│                    └──────────┬──────────┘                         │
│                               │                                     │
│                               ▼                                     │
│                        ┌───────────┐                               │
│                        │  Cluster  │                               │
│                        └───────────┘                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Benefits of GitOps

### Developer Experience

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVELOPER BENEFITS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Familiar Workflow                                           │   │
│  │  • Use Git commands developers already know                  │   │
│  │  • Pull requests for infrastructure changes                  │   │
│  │  • Code review for deployments                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Self-Service Deployments                                    │   │
│  │  • Developers deploy by merging PRs                          │   │
│  │  • No need for cluster credentials                           │   │
│  │  • Reduced ops dependency                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Easy Rollbacks                                              │   │
│  │  • Revert a Git commit to rollback                           │   │
│  │  • Complete history of all changes                           │   │
│  │  • Confidence in disaster recovery                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Operations Benefits

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OPERATIONS BENEFITS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Security                     Reliability                           │
│  ──────────                   ───────────                           │
│  • No cluster creds in CI     • Self-healing systems               │
│  • Audit trail in Git         • Automatic drift correction         │
│  • RBAC via Git permissions   • Consistent environments            │
│  • Signed commits             • Faster recovery                    │
│                                                                     │
│  Compliance                   Observability                         │
│  ──────────                   ─────────────                         │
│  • All changes documented     • Know what's deployed               │
│  • Approval workflows         • Track deployment history           │
│  • SOC2/HIPAA friendly        • Compare desired vs actual          │
│  • Change management          • Real-time sync status              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GitOps Workflow Example

### Day-to-Day Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TYPICAL GITOPS WORKFLOW                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Developer makes changes                                         │
│     ┌────────────────────────────────────────────────────────────┐ │
│     │ $ git checkout -b feature/update-frontend                  │ │
│     │ $ vim apps/production/frontend/deployment.yaml             │ │
│     │ # Change image tag from v1.0.0 to v1.1.0                   │ │
│     │ $ git commit -m "Update frontend to v1.1.0"                │ │
│     │ $ git push origin feature/update-frontend                  │ │
│     └────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  2. Create Pull Request                                             │
│     ┌────────────────────────────────────────────────────────────┐ │
│     │ • Team reviews changes                                     │ │
│     │ • CI runs validation (YAML lint, policy checks)            │ │
│     │ • Security scanning                                        │ │
│     │ • Approval from designated reviewers                       │ │
│     └────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  3. Merge to main branch                                            │
│     ┌────────────────────────────────────────────────────────────┐ │
│     │ • PR merged after approval                                 │ │
│     │ • Git becomes the new source of truth                      │ │
│     └────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  4. GitOps agent detects change                                     │
│     ┌────────────────────────────────────────────────────────────┐ │
│     │ • ArgoCD/Flux polls Git repository                         │ │
│     │ • Detects difference between Git and cluster               │ │
│     │ • Automatically syncs changes                              │ │
│     └────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  5. Cluster updated                                                 │
│     ┌────────────────────────────────────────────────────────────┐ │
│     │ • New frontend pods deployed                               │ │
│     │ • Rolling update performed                                 │ │
│     │ • Sync status: ✓ Synced                                    │ │
│     └────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Common GitOps Patterns

### Environment Branches vs Directories

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ENVIRONMENT PATTERNS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pattern 1: Directory-Based (Recommended)                           │
│  ─────────────────────────────────────────                          │
│  repo/                                                              │
│  ├── environments/                                                  │
│  │   ├── dev/                                                       │
│  │   │   └── kustomization.yaml                                    │
│  │   ├── staging/                                                   │
│  │   │   └── kustomization.yaml                                    │
│  │   └── prod/                                                      │
│  │       └── kustomization.yaml                                    │
│  └── base/                                                          │
│      └── deployment.yaml                                            │
│                                                                     │
│  ✓ Single branch (main)                                            │
│  ✓ Easy to compare environments                                    │
│  ✓ Atomic multi-environment changes                                │
│                                                                     │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                     │
│  Pattern 2: Branch-Based                                            │
│  ───────────────────────                                            │
│  • main branch → production                                        │
│  • staging branch → staging                                        │
│  • develop branch → development                                    │
│                                                                     │
│  ✗ Harder to maintain consistency                                  │
│  ✗ Merge conflicts between environments                            │
│  ✗ Difficult to track what's deployed where                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GitOps Anti-Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GITOPS ANTI-PATTERNS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✗ Manual kubectl Changes                                           │
│    Don't: $ kubectl edit deployment frontend                        │
│    Do: Edit Git, let agent sync                                     │
│                                                                     │
│  ✗ Storing Secrets in Git (Plain Text)                              │
│    Don't: Put passwords in YAML files                               │
│    Do: Use sealed-secrets, SOPS, or external secrets               │
│                                                                     │
│  ✗ CI System Deploying Directly                                     │
│    Don't: CI pipeline runs kubectl apply                            │
│    Do: CI updates Git, GitOps agent deploys                         │
│                                                                     │
│  ✗ Ignoring Drift                                                   │
│    Don't: Allow manual changes to accumulate                        │
│    Do: Enable auto-sync and self-heal                               │
│                                                                     │
│  ✗ Single Repo for Everything                                       │
│    Don't: Mix app code with deployment config                       │
│    Do: Separate app repo from config repo                           │
│                                                                     │
│  ✗ No Validation Before Merge                                       │
│    Don't: Merge without testing manifests                           │
│    Do: Validate YAML, run policy checks in CI                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GITOPS SUMMARY                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  GitOps = IaC + Pull Requests + Continuous Reconciliation           │
│                                                                     │
│  Key Takeaways:                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Git is the single source of truth                         │   │
│  │ • Declarative configuration describes desired state         │   │
│  │ • Agents continuously reconcile actual vs desired           │   │
│  │ • Pull-based model improves security                        │   │
│  │ • All changes are auditable and reversible                  │   │
│  │ • Self-healing infrastructure                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Next: Learn about GitOps tools (ArgoCD, Flux, Jenkins X)          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
