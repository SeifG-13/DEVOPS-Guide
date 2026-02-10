# Threat Modeling

## What is Threat Modeling?

Threat modeling is a structured approach to identifying, quantifying, and addressing security risks.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    THREAT MODELING                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  "Threat modeling is the process of identifying potential          │
│   threats and vulnerabilities BEFORE they can be exploited"        │
│                                                                     │
│  Key Questions:                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. What are we building?                                    │   │
│  │ 2. What can go wrong?                                       │   │
│  │ 3. What are we going to do about it?                        │   │
│  │ 4. Did we do a good job?                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  When to Perform:                                                   │
│  • Design phase of new features                                    │
│  • Architecture changes                                            │
│  • New integrations                                                │
│  • Periodic reviews                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## STRIDE Threat Model

### Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    STRIDE MODEL                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  STRIDE = Microsoft's threat classification model                   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ S │ Spoofing        │ Pretending to be someone else         │   │
│  ├───┼─────────────────┼───────────────────────────────────────┤   │
│  │ T │ Tampering       │ Modifying data or code                │   │
│  ├───┼─────────────────┼───────────────────────────────────────┤   │
│  │ R │ Repudiation     │ Denying having performed an action    │   │
│  ├───┼─────────────────┼───────────────────────────────────────┤   │
│  │ I │ Info Disclosure │ Exposing information to unauthorized  │   │
│  ├───┼─────────────────┼───────────────────────────────────────┤   │
│  │ D │ Denial of Svc   │ Making service unavailable            │   │
│  ├───┼─────────────────┼───────────────────────────────────────┤   │
│  │ E │ Elev. of Priv   │ Gaining unauthorized access           │   │
│  └───┴─────────────────┴───────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### STRIDE Details

```
┌─────────────────────────────────────────────────────────────────────┐
│                    STRIDE IN DETAIL                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SPOOFING                                                           │
│  ─────────                                                          │
│  Threat: Attacker impersonates user, service, or system            │
│  Examples:                                                          │
│  • Stolen credentials                                              │
│  • Session hijacking                                               │
│  • IP spoofing                                                     │
│  • Phishing                                                        │
│  Mitigations:                                                       │
│  • Strong authentication (MFA)                                     │
│  • Certificate validation                                          │
│  • Session management                                              │
│                                                                     │
│  TAMPERING                                                          │
│  ─────────                                                          │
│  Threat: Unauthorized modification of data                         │
│  Examples:                                                          │
│  • SQL injection                                                   │
│  • Man-in-the-middle attacks                                       │
│  • File tampering                                                  │
│  • Memory corruption                                               │
│  Mitigations:                                                       │
│  • Input validation                                                │
│  • Digital signatures                                              │
│  • Integrity checks (hashes)                                       │
│  • Encryption in transit                                           │
│                                                                     │
│  REPUDIATION                                                        │
│  ───────────                                                        │
│  Threat: User denies performing an action                          │
│  Examples:                                                          │
│  • Deleting audit logs                                             │
│  • Denying transactions                                            │
│  • Claiming "I never did that"                                     │
│  Mitigations:                                                       │
│  • Comprehensive logging                                           │
│  • Audit trails                                                    │
│  • Digital signatures                                              │
│  • Non-repudiation mechanisms                                      │
│                                                                     │
│  INFORMATION DISCLOSURE                                             │
│  ──────────────────────                                             │
│  Threat: Unauthorized access to sensitive data                     │
│  Examples:                                                          │
│  • Data breaches                                                   │
│  • Error messages revealing info                                   │
│  • Insecure storage                                                │
│  • Network sniffing                                                │
│  Mitigations:                                                       │
│  • Encryption (at rest and in transit)                             │
│  • Access controls                                                 │
│  • Data masking                                                    │
│  • Secure error handling                                           │
│                                                                     │
│  DENIAL OF SERVICE                                                  │
│  ─────────────────                                                  │
│  Threat: Making services unavailable                               │
│  Examples:                                                          │
│  • DDoS attacks                                                    │
│  • Resource exhaustion                                             │
│  • Application crashes                                             │
│  • Disk filling                                                    │
│  Mitigations:                                                       │
│  • Rate limiting                                                   │
│  • Input validation                                                │
│  • Resource quotas                                                 │
│  • Redundancy/failover                                             │
│                                                                     │
│  ELEVATION OF PRIVILEGE                                             │
│  ──────────────────────                                             │
│  Threat: Gaining higher access than authorized                     │
│  Examples:                                                          │
│  • Privilege escalation exploits                                   │
│  • Broken access controls                                          │
│  • SQL injection to admin                                          │
│  • Container escapes                                               │
│  Mitigations:                                                       │
│  • Principle of least privilege                                    │
│  • Input validation                                                │
│  • Secure coding practices                                         │
│  • Regular patching                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## DREAD Risk Assessment

### DREAD Scoring

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DREAD MODEL                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DREAD = Risk rating model (score 1-10 each)                       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ D │ Damage        │ How bad is the impact?                  │   │
│  │   │               │ 1=minimal, 10=catastrophic               │   │
│  ├───┼───────────────┼─────────────────────────────────────────┤   │
│  │ R │ Reproducibility│ How easy to reproduce?                 │   │
│  │   │               │ 1=very hard, 10=always works             │   │
│  ├───┼───────────────┼─────────────────────────────────────────┤   │
│  │ E │ Exploitability│ How easy to exploit?                    │   │
│  │   │               │ 1=expert only, 10=anyone                 │   │
│  ├───┼───────────────┼─────────────────────────────────────────┤   │
│  │ A │ Affected Users│ How many users affected?                │   │
│  │   │               │ 1=few, 10=all users                      │   │
│  ├───┼───────────────┼─────────────────────────────────────────┤   │
│  │ D │ Discoverability│ How easy to discover?                  │   │
│  │   │               │ 1=very hard, 10=obvious                  │   │
│  └───┴───────────────┴─────────────────────────────────────────┘   │
│                                                                     │
│  Risk Score = (D + R + E + A + D) / 5                              │
│                                                                     │
│  Rating Scale:                                                      │
│  • 1-3: Low risk                                                   │
│  • 4-6: Medium risk                                                │
│  • 7-10: High risk                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### DREAD Example

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DREAD EXAMPLE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Threat: SQL Injection in Login Form                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Damage         │ 9 │ Full database access                   │   │
│  │ Reproducibility│ 8 │ Easy to reproduce with tools           │   │
│  │ Exploitability │ 7 │ Many automated tools available         │   │
│  │ Affected Users │ 10│ All users' data at risk                │   │
│  │ Discoverability│ 8 │ Common attack vector, easily found     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Risk Score = (9 + 8 + 7 + 10 + 8) / 5 = 8.4 (HIGH RISK)          │
│                                                                     │
│  Priority: CRITICAL - Fix immediately                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Attack Surface Analysis

### Identifying Attack Surface

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ATTACK SURFACE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Attack Surface = All points where attacker can interact           │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    APPLICATION                               │   │
│  │                                                              │   │
│  │  Entry Points:              Data Elements:                   │   │
│  │  ┌───────────────┐         ┌───────────────┐                │   │
│  │  │ • Web forms   │         │ • User input  │                │   │
│  │  │ • APIs        │         │ • Cookies     │                │   │
│  │  │ • File uploads│         │ • Sessions    │                │   │
│  │  │ • URLs/params │         │ • Database    │                │   │
│  │  │ • WebSockets  │         │ • Files       │                │   │
│  │  │ • Message Qs  │         │ • Env vars    │                │   │
│  │  └───────────────┘         └───────────────┘                │   │
│  │                                                              │   │
│  │  External Services:        Trust Boundaries:                 │   │
│  │  ┌───────────────┐         ┌───────────────┐                │   │
│  │  │ • Third-party │         │ • Network     │                │   │
│  │  │ • Cloud       │         │ • Process     │                │   │
│  │  │ • Database    │         │ • User levels │                │   │
│  │  │ • Auth        │         │ • Machine     │                │   │
│  │  └───────────────┘         └───────────────┘                │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Attack Surface Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ATTACK SURFACE CHECKLIST                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Network:                                                           │
│  □ Open ports                                                      │
│  □ Network protocols                                               │
│  □ Load balancers                                                  │
│  □ DNS configuration                                               │
│  □ TLS/SSL configuration                                           │
│                                                                     │
│  Application:                                                       │
│  □ Authentication endpoints                                        │
│  □ API endpoints                                                   │
│  □ File upload functionality                                       │
│  □ Admin interfaces                                                │
│  □ Error handling/messages                                         │
│                                                                     │
│  Data:                                                              │
│  □ Database connections                                            │
│  □ Session management                                              │
│  □ Encryption usage                                                │
│  □ Sensitive data storage                                          │
│  □ Backup locations                                                │
│                                                                     │
│  Infrastructure:                                                    │
│  □ Container configurations                                        │
│  □ Kubernetes RBAC                                                 │
│  □ Cloud IAM                                                       │
│  □ Secrets management                                              │
│  □ Logging/monitoring                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Diagrams

### Creating DFDs

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DATA FLOW DIAGRAM ELEMENTS                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Elements:                                                          │
│                                                                     │
│  ┌─────────────┐    External Entity (user, external system)        │
│  │   Entity    │    Outside system boundary                        │
│  └─────────────┘                                                    │
│                                                                     │
│  (   Process   )    Process (transforms data)                      │
│                     Part of the system                              │
│                                                                     │
│  ═══════════════    Data Store (database, file)                    │
│  ║ Data Store ║    Persists data                                   │
│  ═══════════════                                                    │
│                                                                     │
│  ─────────────▶     Data Flow                                      │
│                     Shows data movement                             │
│                                                                     │
│  - - - - - - - -    Trust Boundary                                 │
│                     Separates trust levels                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Example DFD

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WEB APPLICATION DFD                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐                                                      │
│  │   User   │                                                      │
│  │(Browser) │                                                      │
│  └────┬─────┘                                                      │
│       │                                                             │
│       │ HTTPS Request                                               │
│       ▼                                                             │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ TRUST BOUNDARY ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │
│       │                                                             │
│       ▼                                                             │
│  (  Load     )                                                     │
│  ( Balancer  )                                                     │
│       │                                                             │
│       │ HTTP                                                        │
│       ▼                                                             │
│  (   Web     )──────────▶ ═══════════════                          │
│  (  Server   )  Session   ║   Redis     ║                          │
│  (   API     )◀────────── ║   Cache     ║                          │
│       │                   ═══════════════                          │
│       │ SQL Query                                                   │
│       ▼                                                             │
│  ═══════════════          ┌──────────────┐                         │
│  ║  Database  ║◀─────────▶│   External   │                         │
│  ║ (PostgreSQL)║  API Call│    Payment   │                         │
│  ═══════════════          │    Service   │                         │
│                           └──────────────┘                         │
│                                                                     │
│  Threats to Analyze at Each Point:                                  │
│  • User → LB: MITM, spoofing                                       │
│  • LB → Web: Internal network attacks                              │
│  • Web → DB: SQL injection, data exposure                          │
│  • Web → External: API key exposure, data leakage                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Threat Modeling Process

### Step-by-Step Process

```
┌─────────────────────────────────────────────────────────────────────┐
│                    THREAT MODELING PROCESS                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Step 1: DECOMPOSE THE APPLICATION                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Identify entry points                                     │   │
│  │ • Identify assets                                           │   │
│  │ • Identify trust levels                                     │   │
│  │ • Create data flow diagrams                                 │   │
│  │ • Document external dependencies                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Step 2: IDENTIFY THREATS                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Apply STRIDE to each component                            │   │
│  │ • Consider attack vectors                                   │   │
│  │ • Review common vulnerabilities                             │   │
│  │ • Brainstorm with team                                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Step 3: RANK THREATS                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Apply DREAD scoring                                       │   │
│  │ • Prioritize by risk                                        │   │
│  │ • Consider business impact                                  │   │
│  │ • Create risk matrix                                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Step 4: DETERMINE MITIGATIONS                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Identify countermeasures                                  │   │
│  │ • Evaluate mitigation effectiveness                         │   │
│  │ • Consider cost vs. benefit                                 │   │
│  │ • Document security controls                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Step 5: VALIDATE                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Review with stakeholders                                  │   │
│  │ • Test mitigations                                          │   │
│  │ • Update threat model                                       │   │
│  │ • Track findings                                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Threat Modeling Tools

```
┌─────────────────────────────────────────────────────────────────────┐
│                    THREAT MODELING TOOLS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Free Tools:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ OWASP Threat Dragon  │ Web-based, STRIDE support            │   │
│  │ Microsoft TMT        │ Windows desktop, DFD diagrams        │   │
│  │ OWASP pytm           │ Python code-based threat modeling    │   │
│  │ Threagile            │ YAML-based, automated analysis       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Commercial Tools:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ IriusRisk            │ Enterprise, CI/CD integration        │   │
│  │ ThreatModeler        │ Cloud-native, automated              │   │
│  │ SD Elements          │ Requirements-based security          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Threagile Example

```yaml
# threagile.yaml - Threat model as code
title: My Application

data_assets:
  customer-data:
    description: Customer PII
    usage: business
    quantity: many
    confidentiality: confidential
    integrity: critical
    availability: important

technical_assets:
  web-server:
    description: Frontend web server
    type: process
    usage: business
    technologies:
      - web-server
    data_assets_processed:
      - customer-data
    communication_links:
      database-connection:
        target: database
        protocol: tcp
        authentication: credentials

  database:
    description: PostgreSQL database
    type: datastore
    technologies:
      - database
    data_assets_stored:
      - customer-data
```

---

## Threat Modeling Template

```
┌─────────────────────────────────────────────────────────────────────┐
│                    THREAT MODEL DOCUMENT                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Project: [Name]                     Date: [Date]                  │
│  Version: [1.0]                      Author: [Name]                │
│                                                                     │
│  1. SYSTEM DESCRIPTION                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ [Describe the system, its purpose, and scope]               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  2. ASSETS                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • [Asset 1: Description, sensitivity level]                 │   │
│  │ • [Asset 2: Description, sensitivity level]                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  3. DATA FLOW DIAGRAM                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ [Insert DFD here]                                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  4. THREATS                                                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ID  │ Threat      │ STRIDE │ DREAD │ Mitigation │ Status    │  │
│  ├──────────────────────────────────────────────────────────────┤  │
│  │ T1  │ SQL Inject  │ T,I,E  │ 8.4   │ Prepared   │ Mitigated │  │
│  │ T2  │ CSRF        │ S      │ 6.2   │ Tokens     │ Open      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  5. SECURITY CONTROLS                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • [Control 1: Description, threats addressed]               │   │
│  │ • [Control 2: Description, threats addressed]               │   │
│  └─────────────────────────────────────────────────────────────┘   │
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
│  Key Frameworks:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ STRIDE │ Threat classification (what can go wrong)          │   │
│  │ DREAD  │ Risk scoring (how bad is it)                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Process Steps:                                                     │
│  1. Decompose application (DFDs, assets, trust boundaries)         │
│  2. Identify threats (apply STRIDE)                                │
│  3. Rank threats (DREAD scoring)                                   │
│  4. Determine mitigations                                          │
│  5. Validate and iterate                                           │
│                                                                     │
│  Best Practices:                                                    │
│  • Do threat modeling early in design                              │
│  • Involve the whole team                                          │
│  • Update when architecture changes                                │
│  • Track threats as security requirements                          │
│  • Automate where possible (Threagile, pytm)                       │
│                                                                     │
│  Next: Learn secure coding practices                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
