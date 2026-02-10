# Compliance and Auditing

## Compliance Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COMPLIANCE IN DEVSECOPS                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What is Compliance?                                                │
│  • Adhering to regulatory requirements and standards               │
│  • Demonstrating security controls are in place                    │
│  • Maintaining audit trails and evidence                           │
│  • Continuous monitoring and reporting                             │
│                                                                     │
│  Common Compliance Frameworks:                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ SOC 2        │ Service organization controls                │   │
│  │ PCI DSS      │ Payment card industry                        │   │
│  │ HIPAA        │ Healthcare data protection                   │   │
│  │ GDPR         │ European data protection                     │   │
│  │ ISO 27001    │ Information security management              │   │
│  │ FedRAMP      │ US federal cloud security                    │   │
│  │ CIS          │ Security configuration benchmarks            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Compliance as Code

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COMPLIANCE AS CODE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Traditional vs Automated Compliance:                               │
│                                                                     │
│  Traditional:                                                       │
│  • Manual audits (annual/quarterly)                                │
│  • Point-in-time snapshots                                         │
│  • Document-heavy process                                          │
│  • Reactive remediation                                            │
│                                                                     │
│  Compliance as Code:                                                │
│  • Automated continuous checks                                     │
│  • Real-time compliance status                                     │
│  • Policy defined in code                                          │
│  • Proactive prevention                                            │
│                                                                     │
│  Benefits:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Speed         │ Instant compliance verification             │   │
│  │ Consistency   │ Same checks every time                      │   │
│  │ Auditability  │ Version-controlled policies                 │   │
│  │ Scalability   │ Works across environments                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Open Policy Agent (OPA)

### Installation

```bash
# Install OPA
curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64
chmod +x opa
sudo mv opa /usr/local/bin/

# Install Gatekeeper on Kubernetes
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

### OPA Policies (Rego)

```rego
# policy/kubernetes.rego
package kubernetes.admission

# Deny privileged containers
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    container.securityContext.privileged == true
    msg := sprintf("Privileged containers not allowed: %v", [container.name])
}

# Require resource limits
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    not container.resources.limits.memory
    msg := sprintf("Memory limits required for container: %v", [container.name])
}

# Enforce image registry
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    not startswith(container.image, "gcr.io/mycompany/")
    not startswith(container.image, "docker.io/library/")
    msg := sprintf("Image from untrusted registry: %v", [container.image])
}

# Require labels
deny[msg] {
    input.request.kind.kind == "Pod"
    not input.request.object.metadata.labels.app
    msg := "Pod must have 'app' label"
}

# Deny default namespace
deny[msg] {
    input.request.kind.kind == "Pod"
    input.request.object.metadata.namespace == "default"
    msg := "Pods cannot be deployed to 'default' namespace"
}
```

### Gatekeeper Constraint Templates

```yaml
# ConstraintTemplate - Require Labels
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
---
# Constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - production
  parameters:
    labels:
      - "app"
      - "environment"
      - "owner"
```

### Gatekeeper - Container Security

```yaml
# ConstraintTemplate - No Privileged
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spsprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spsprivilegedcontainer

        violation[{"msg": msg}] {
          c := input_containers[_]
          c.securityContext.privileged
          msg := sprintf("Privileged container not allowed: %v", [c.name])
        }

        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }
        input_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: deny-privileged-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
```

---

## Audit Logging

### Kubernetes Audit Policy

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log authentication at metadata level
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
    verbs: ["get", "list", "watch"]

  # Log all changes to pods at request level
  - level: Request
    resources:
      - group: ""
        resources: ["pods"]
    verbs: ["create", "update", "patch", "delete"]

  # Log exec/attach at RequestResponse level
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]

  # Log secret access
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]

  # Log role bindings changes
  - level: Request
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["rolebindings", "clusterrolebindings"]
    verbs: ["create", "update", "patch", "delete"]

  # Don't log read-only endpoints
  - level: None
    nonResourceURLs:
      - "/healthz*"
      - "/readyz*"
      - "/livez*"
      - "/metrics"

  # Default catch-all
  - level: Metadata
```

### AWS CloudTrail

```hcl
# Terraform - CloudTrail configuration
resource "aws_cloudtrail" "main" {
  name                          = "main-trail"
  s3_bucket_name               = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail        = true
  enable_logging               = true

  # CloudWatch integration
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail.arn

  # Enable log file validation
  enable_log_file_validation = true

  # Event selectors for management events
  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }

  # Data events for S3
  event_selector {
    read_write_type           = "All"
    include_management_events = false

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::sensitive-bucket/"]
    }
  }

  # KMS encryption
  kms_key_id = aws_kms_key.cloudtrail.arn

  tags = {
    Environment = "production"
    Compliance  = "required"
  }
}

# CloudWatch alerts for suspicious activity
resource "aws_cloudwatch_metric_alarm" "unauthorized_api" {
  alarm_name          = "unauthorized-api-calls"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "UnauthorizedAttemptCount"
  namespace           = "CloudTrailMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "Unauthorized API calls detected"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

### Audit Log Analysis

```yaml
# Elasticsearch index template for audit logs
PUT _index_template/audit-logs
{
  "index_patterns": ["audit-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "audit-policy",
      "index.lifecycle.rollover_alias": "audit"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "user": {
          "properties": {
            "username": { "type": "keyword" },
            "groups": { "type": "keyword" }
          }
        },
        "verb": { "type": "keyword" },
        "resource": { "type": "keyword" },
        "namespace": { "type": "keyword" },
        "responseStatus": {
          "properties": {
            "code": { "type": "integer" }
          }
        },
        "sourceIPs": { "type": "ip" }
      }
    }
  }
}
```

---

## Compliance Scanning

### Kube-bench (CIS Kubernetes)

```bash
# Run CIS benchmark
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Get results
kubectl logs job.batch/kube-bench

# Run specific checks
kube-bench run --targets master
kube-bench run --targets node
kube-bench run --targets etcd
```

```yaml
# Kube-bench Job
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    spec:
      hostPID: true
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:latest
          command: ["kube-bench"]
          volumeMounts:
            - name: var-lib-etcd
              mountPath: /var/lib/etcd
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: var-lib-etcd
          hostPath:
            path: /var/lib/etcd
        - name: etc-kubernetes
          hostPath:
            path: /etc/kubernetes
```

### Polaris (Kubernetes Best Practices)

```bash
# Install Polaris
kubectl apply -f https://github.com/fairwindsops/polaris/releases/latest/download/dashboard.yaml

# Port forward dashboard
kubectl port-forward -n polaris svc/polaris-dashboard 8080:80

# CLI scan
polaris audit --audit-path ./k8s/ --format yaml
```

```yaml
# Polaris configuration
checks:
  # Security
  hostNetworkSet: danger
  hostPIDSet: danger
  hostIPCSet: danger
  privilegeEscalationAllowed: danger
  runAsRootAllowed: danger
  runAsPrivileged: danger
  notReadOnlyRootFilesystem: warning

  # Resources
  cpuRequestsMissing: warning
  cpuLimitsMissing: warning
  memoryRequestsMissing: warning
  memoryLimitsMissing: warning

  # Images
  tagNotSpecified: danger
  pullPolicyNotAlways: warning

  # Health
  readinessProbeMissing: warning
  livenessProbeMissing: warning

exemptions:
  - namespace: kube-system
    controllerNames:
      - kube-proxy
    rules:
      - hostNetworkSet
```

### InSpec (Infrastructure Compliance)

```ruby
# controls/kubernetes.rb
control 'k8s-1.1' do
  impact 1.0
  title 'Ensure API server anonymous auth is disabled'
  desc 'Anonymous requests should be disabled on the API server'

  describe command('kubectl get pods -n kube-system -l component=kube-apiserver -o yaml') do
    its('stdout') { should include '--anonymous-auth=false' }
  end
end

control 'k8s-1.2' do
  impact 1.0
  title 'Ensure RBAC is enabled'
  desc 'RBAC should be enabled for authorization'

  describe command('kubectl api-versions') do
    its('stdout') { should include 'rbac.authorization.k8s.io' }
  end
end

control 'k8s-2.1' do
  impact 0.7
  title 'Ensure namespaces are used'
  desc 'Do not use the default namespace'

  describe command('kubectl get pods -n default') do
    its('stdout') { should_not match(/Running|Pending/) }
  end
end
```

```bash
# Run InSpec
inspec exec ./compliance/kubernetes --reporter cli json:results.json
```

---

## Compliance Reporting

### Automated Compliance Dashboard

```yaml
# Grafana dashboard for compliance metrics
apiVersion: v1
kind: ConfigMap
metadata:
  name: compliance-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  compliance.json: |
    {
      "title": "Compliance Dashboard",
      "panels": [
        {
          "title": "Policy Violations by Severity",
          "type": "piechart",
          "targets": [
            {
              "expr": "sum(gatekeeper_violations) by (enforcement_action)",
              "legendFormat": "{{enforcement_action}}"
            }
          ]
        },
        {
          "title": "CIS Benchmark Score",
          "type": "gauge",
          "targets": [
            {
              "expr": "kube_bench_pass_total / (kube_bench_pass_total + kube_bench_fail_total) * 100"
            }
          ]
        },
        {
          "title": "Compliance Trend",
          "type": "timeseries",
          "targets": [
            {
              "expr": "sum(gatekeeper_violations) by (namespace)",
              "legendFormat": "{{namespace}}"
            }
          ]
        }
      ]
    }
```

### Compliance Report Generation

```python
#!/usr/bin/env python3
# compliance_report.py
import json
from datetime import datetime
import subprocess

def run_compliance_checks():
    results = {
        "timestamp": datetime.utcnow().isoformat(),
        "checks": []
    }

    # Kube-bench
    kube_bench = subprocess.run(
        ["kube-bench", "--json"],
        capture_output=True, text=True
    )
    results["kube_bench"] = json.loads(kube_bench.stdout)

    # Gatekeeper violations
    violations = subprocess.run(
        ["kubectl", "get", "constraints", "-o", "json"],
        capture_output=True, text=True
    )
    results["gatekeeper"] = json.loads(violations.stdout)

    # Polaris audit
    polaris = subprocess.run(
        ["polaris", "audit", "--format", "json"],
        capture_output=True, text=True
    )
    results["polaris"] = json.loads(polaris.stdout)

    return results

def generate_report(results):
    report = f"""
# Compliance Report
Generated: {results['timestamp']}

## CIS Kubernetes Benchmark
- Pass: {results['kube_bench']['Totals']['total_pass']}
- Fail: {results['kube_bench']['Totals']['total_fail']}
- Warn: {results['kube_bench']['Totals']['total_warn']}

## Policy Violations
Total Violations: {len(results['gatekeeper'].get('items', []))}

## Polaris Score
Score: {results['polaris'].get('score', 'N/A')}
    """
    return report

if __name__ == "__main__":
    results = run_compliance_checks()
    report = generate_report(results)
    print(report)
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                 COMPLIANCE BEST PRACTICES                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Policy Management:                                                 │
│  ✓ Define policies as code (OPA, Gatekeeper)                       │
│  ✓ Version control all policies                                    │
│  ✓ Test policies before deployment                                 │
│  ✓ Use dry-run/audit mode initially                                │
│                                                                     │
│  Audit Logging:                                                     │
│  ✓ Enable comprehensive audit logging                              │
│  ✓ Protect log integrity (immutable storage)                       │
│  ✓ Retain logs per compliance requirements                         │
│  ✓ Set up alerts for suspicious activity                           │
│                                                                     │
│  Continuous Compliance:                                             │
│  ✓ Automate compliance scans in CI/CD                              │
│  ✓ Generate regular compliance reports                             │
│  ✓ Track compliance metrics over time                              │
│  ✓ Remediate findings promptly                                     │
│                                                                     │
│  Documentation:                                                     │
│  ✓ Maintain evidence of controls                                   │
│  ✓ Document exceptions and compensating controls                   │
│  ✓ Keep audit trail of policy changes                              │
│  ✓ Regular review and update of policies                           │
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
│  Compliance Tools:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ OPA/Gatekeeper│ Policy enforcement in Kubernetes            │   │
│  │ Kube-bench    │ CIS Kubernetes benchmark                    │   │
│  │ Polaris       │ Kubernetes best practices                   │   │
│  │ InSpec        │ Infrastructure compliance testing           │   │
│  │ CloudTrail    │ AWS API audit logging                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Concepts:                                                      │
│  • Compliance as Code - automate everything                        │
│  • Continuous compliance - not point-in-time                       │
│  • Audit logging - comprehensive and protected                     │
│  • Policy enforcement - prevent violations proactively             │
│                                                                     │
│  Compliance Workflow:                                               │
│  1. Define policies in code                                        │
│  2. Enforce at admission (Gatekeeper)                              │
│  3. Scan continuously (kube-bench, Polaris)                        │
│  4. Log and monitor (audit logs, alerts)                           │
│  5. Report and remediate                                           │
│                                                                     │
│  Next: Learn about incident response                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
