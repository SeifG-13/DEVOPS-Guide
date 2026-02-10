# Incident Response

## Incident Response Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INCIDENT RESPONSE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What is Incident Response?                                        │
│  • Organized approach to handling security incidents               │
│  • Minimizing damage and recovery time                             │
│  • Learning from incidents to prevent recurrence                   │
│  • Maintaining business continuity                                 │
│                                                                     │
│  Incident Types:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Security Breach │ Unauthorized access to systems            │   │
│  │ Data Leak       │ Exposure of sensitive data                │   │
│  │ Malware         │ Infection of systems                      │   │
│  │ DDoS            │ Service disruption attack                 │   │
│  │ Insider Threat  │ Malicious internal actor                  │   │
│  │ Supply Chain    │ Compromised dependencies                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Incident Response Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IR LIFECYCLE (NIST)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐               │
│  │ Preparation│───▶│ Detection  │───▶│Containment │               │
│  │            │    │ & Analysis │    │            │               │
│  └────────────┘    └────────────┘    └─────┬──────┘               │
│        ▲                                    │                       │
│        │                                    ▼                       │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐               │
│  │ Post-      │◀───│  Recovery  │◀───│Eradication │               │
│  │ Incident   │    │            │    │            │               │
│  └────────────┘    └────────────┘    └────────────┘               │
│                                                                     │
│  1. Preparation     │ Tools, training, playbooks                   │
│  2. Detection       │ Identify and analyze incidents               │
│  3. Containment     │ Limit damage and spread                      │
│  4. Eradication     │ Remove threat completely                     │
│  5. Recovery        │ Restore normal operations                    │
│  6. Post-Incident   │ Learn and improve                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Incident Severity Levels

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SEVERITY CLASSIFICATION                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SEV-1 (Critical):                                                  │
│  • Active data breach in progress                                  │
│  • Production systems compromised                                  │
│  • Customer data exposed                                           │
│  Response: Immediate, all-hands, executive notification            │
│                                                                     │
│  SEV-2 (High):                                                      │
│  • Potential breach detected                                       │
│  • Security vulnerability actively exploited                       │
│  • Critical infrastructure affected                                │
│  Response: Within 1 hour, security team mobilized                  │
│                                                                     │
│  SEV-3 (Medium):                                                    │
│  • Suspicious activity detected                                    │
│  • Non-critical vulnerability found                                │
│  • Security policy violation                                       │
│  Response: Within 4 hours, investigate and remediate               │
│                                                                     │
│  SEV-4 (Low):                                                       │
│  • Minor security misconfiguration                                 │
│  • Failed attack attempt blocked                                   │
│  • Routine security alert                                          │
│  Response: Within 24 hours, document and fix                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Incident Response Playbooks

### Compromised Container Playbook

```yaml
# playbook-compromised-container.yaml
name: Compromised Container Response
severity: SEV-1
triggers:
  - Falco critical alert
  - Unexpected process execution
  - Cryptocurrency miner detected

steps:
  detection:
    - Verify alert is not false positive
    - Identify affected pod/container
    - Determine scope of compromise
    - Collect initial evidence

  containment:
    immediate:
      - Isolate pod with network policy
      - Capture pod state for forensics
      - Prevent spread to other pods
    commands:
      - name: Apply network isolation
        cmd: |
          kubectl apply -f - <<EOF
          apiVersion: networking.k8s.io/v1
          kind: NetworkPolicy
          metadata:
            name: isolate-compromised
            namespace: ${NAMESPACE}
          spec:
            podSelector:
              matchLabels:
                app: ${APP_NAME}
            policyTypes:
              - Ingress
              - Egress
          EOF

      - name: Capture pod state
        cmd: |
          kubectl describe pod ${POD_NAME} -n ${NAMESPACE} > pod-describe.txt
          kubectl logs ${POD_NAME} -n ${NAMESPACE} --all-containers > pod-logs.txt
          kubectl get events -n ${NAMESPACE} --field-selector involvedObject.name=${POD_NAME} > pod-events.txt

  eradication:
    - Delete compromised pod
    - Scan image for vulnerabilities
    - Review deployment configuration
    - Check for persistence mechanisms
    commands:
      - name: Delete compromised pod
        cmd: kubectl delete pod ${POD_NAME} -n ${NAMESPACE} --grace-period=0 --force

      - name: Scan image
        cmd: trivy image ${IMAGE_NAME}

  recovery:
    - Deploy clean version
    - Verify no persistence
    - Monitor for recurrence
    - Update image to patched version

  post_incident:
    - Document timeline
    - Root cause analysis
    - Update security policies
    - Team debrief
```

### Secrets Exposure Playbook

```yaml
# playbook-secrets-exposed.yaml
name: Secrets Exposure Response
severity: SEV-1
triggers:
  - Secret detected in git commit
  - TruffleHog alert
  - Credential found in logs

steps:
  detection:
    - Identify exposed secrets
    - Determine exposure scope (public/private)
    - Assess potential impact
    - Document affected systems

  containment:
    immediate:
      - Rotate exposed credentials immediately
      - Revoke compromised API keys
      - Disable compromised accounts
    commands:
      - name: Rotate AWS credentials
        cmd: |
          aws iam create-access-key --user-name ${USER_NAME}
          aws iam delete-access-key --user-name ${USER_NAME} --access-key-id ${OLD_KEY}

      - name: Rotate Kubernetes secret
        cmd: |
          kubectl create secret generic ${SECRET_NAME} \
            --from-literal=key=${NEW_VALUE} \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Invalidate tokens
        cmd: |
          vault token revoke ${TOKEN}

  eradication:
    - Remove secrets from git history
    - Clean build caches
    - Rotate all potentially affected secrets
    commands:
      - name: Remove from git history
        cmd: |
          git filter-branch --force --index-filter \
            "git rm --cached --ignore-unmatch ${FILE_PATH}" \
            --prune-empty --tag-name-filter cat -- --all
          git push origin --force --all

  recovery:
    - Deploy with new credentials
    - Verify systems functioning
    - Enable secret scanning
    - Update CI/CD pipelines

  post_incident:
    - Audit secret management practices
    - Implement pre-commit hooks
    - Train developers on secret handling
```

---

## Automated Response

### Kubernetes-based Response

```yaml
# Automated pod isolation on Falco alert
apiVersion: v1
kind: ConfigMap
metadata:
  name: incident-response-scripts
  namespace: security
data:
  isolate-pod.sh: |
    #!/bin/bash
    POD_NAME=$1
    NAMESPACE=$2

    # Apply network isolation
    cat <<EOF | kubectl apply -f -
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: isolate-${POD_NAME}
      namespace: ${NAMESPACE}
    spec:
      podSelector:
        matchLabels:
          isolation: quarantine
      policyTypes:
        - Ingress
        - Egress
    EOF

    # Label pod for isolation
    kubectl label pod ${POD_NAME} -n ${NAMESPACE} isolation=quarantine

    # Capture forensics
    kubectl describe pod ${POD_NAME} -n ${NAMESPACE} > /evidence/${POD_NAME}-describe.txt
    kubectl logs ${POD_NAME} -n ${NAMESPACE} --all-containers > /evidence/${POD_NAME}-logs.txt

    # Notify security team
    curl -X POST ${SLACK_WEBHOOK} \
      -H 'Content-type: application/json' \
      -d "{\"text\":\"Pod ${POD_NAME} in ${NAMESPACE} has been isolated\"}"
---
# Falco response integration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: falco-responder
  namespace: security
spec:
  replicas: 1
  selector:
    matchLabels:
      app: falco-responder
  template:
    spec:
      serviceAccountName: incident-responder
      containers:
        - name: responder
          image: incident-responder:latest
          env:
            - name: FALCO_GRPC_ADDR
              value: falco:5060
            - name: SLACK_WEBHOOK
              valueFrom:
                secretKeyRef:
                  name: slack-webhook
                  key: url
```

### AWS Lambda Response

```python
# lambda_function.py - Automated GuardDuty response
import boto3
import json

def lambda_handler(event, context):
    finding = event['detail']
    severity = finding['severity']
    finding_type = finding['type']

    # High severity findings - immediate action
    if severity >= 7.0:
        handle_critical_finding(finding)

    # Credential compromise
    if 'UnauthorizedAccess:IAMUser' in finding_type:
        handle_credential_compromise(finding)

    # Crypto mining
    if 'CryptoCurrency' in finding_type:
        handle_crypto_mining(finding)

    return {'statusCode': 200}

def handle_credential_compromise(finding):
    iam = boto3.client('iam')
    user_name = finding['resource']['accessKeyDetails']['userName']

    # Deactivate access keys
    keys = iam.list_access_keys(UserName=user_name)
    for key in keys['AccessKeyMetadata']:
        iam.update_access_key(
            UserName=user_name,
            AccessKeyId=key['AccessKeyId'],
            Status='Inactive'
        )

    # Notify security team
    notify_security_team(f"Deactivated keys for user {user_name}")

def handle_crypto_mining(finding):
    ec2 = boto3.client('ec2')
    instance_id = finding['resource']['instanceDetails']['instanceId']

    # Isolate instance
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=['sg-isolated']  # Isolated security group
    )

    # Stop instance
    ec2.stop_instances(InstanceIds=[instance_id])

    notify_security_team(f"Isolated and stopped instance {instance_id}")

def notify_security_team(message):
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:region:account:security-alerts',
        Message=message,
        Subject='Security Incident Alert'
    )
```

---

## Forensics Collection

### Container Forensics

```bash
#!/bin/bash
# container-forensics.sh

POD_NAME=$1
NAMESPACE=$2
EVIDENCE_DIR="/evidence/$(date +%Y%m%d_%H%M%S)_${POD_NAME}"

mkdir -p ${EVIDENCE_DIR}

echo "Collecting forensics for pod ${POD_NAME} in ${NAMESPACE}"

# Get pod details
kubectl describe pod ${POD_NAME} -n ${NAMESPACE} > ${EVIDENCE_DIR}/pod-describe.txt

# Get container logs
kubectl logs ${POD_NAME} -n ${NAMESPACE} --all-containers > ${EVIDENCE_DIR}/container-logs.txt

# Get events
kubectl get events -n ${NAMESPACE} --field-selector involvedObject.name=${POD_NAME} > ${EVIDENCE_DIR}/events.txt

# Get pod YAML
kubectl get pod ${POD_NAME} -n ${NAMESPACE} -o yaml > ${EVIDENCE_DIR}/pod-yaml.txt

# Get network policies
kubectl get networkpolicy -n ${NAMESPACE} -o yaml > ${EVIDENCE_DIR}/netpol.txt

# Capture running processes (if possible)
CONTAINER_ID=$(kubectl get pod ${POD_NAME} -n ${NAMESPACE} -o jsonpath='{.status.containerStatuses[0].containerID}' | sed 's/.*:\/\///')

# If using containerd
crictl inspect ${CONTAINER_ID} > ${EVIDENCE_DIR}/container-inspect.txt 2>/dev/null

# Copy container filesystem
kubectl cp ${NAMESPACE}/${POD_NAME}:/ ${EVIDENCE_DIR}/filesystem/ 2>/dev/null || echo "Filesystem copy failed"

# Create evidence hash
find ${EVIDENCE_DIR} -type f -exec sha256sum {} \; > ${EVIDENCE_DIR}/hashes.txt

echo "Forensics collected in ${EVIDENCE_DIR}"
```

### Memory and Disk Forensics

```yaml
# forensics-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: forensics-collection
  namespace: security
spec:
  template:
    spec:
      hostPID: true
      hostNetwork: true
      nodeSelector:
        kubernetes.io/hostname: ${AFFECTED_NODE}
      containers:
        - name: forensics
          image: forensics-toolkit:latest
          securityContext:
            privileged: true
          command: ["/bin/sh", "-c"]
          args:
            - |
              # Capture memory dump
              dd if=/dev/mem of=/evidence/memory.dump bs=1M count=1024

              # List all processes
              ps auxf > /evidence/processes.txt

              # Network connections
              netstat -tulpn > /evidence/network.txt
              ss -tulpn >> /evidence/network.txt

              # Open files
              lsof > /evidence/open-files.txt

              # Mounted filesystems
              mount > /evidence/mounts.txt
              df -h >> /evidence/mounts.txt

              # Running containers
              crictl ps -a > /evidence/containers.txt
              crictl images > /evidence/images.txt

              # System logs
              journalctl --since "1 hour ago" > /evidence/journal.txt

              # Create hash of evidence
              sha256sum /evidence/* > /evidence/hashes.txt
          volumeMounts:
            - name: evidence
              mountPath: /evidence
            - name: host-root
              mountPath: /host
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: evidence
          persistentVolumeClaim:
            claimName: forensics-evidence
        - name: host-root
          hostPath:
            path: /
```

---

## Communication Templates

### Initial Notification

```markdown
# Security Incident Notification

**Incident ID:** INC-2024-001
**Severity:** SEV-1 (Critical)
**Status:** Active - Containment in Progress
**Time Detected:** 2024-01-15 14:30 UTC

## Summary
A potential security incident has been detected involving [brief description].

## Impact
- Systems affected: [list]
- Data potentially exposed: [description]
- Business impact: [description]

## Current Actions
- [ ] Incident response team activated
- [ ] Containment measures implemented
- [ ] Investigation in progress

## Next Update
Expected in 30 minutes or when significant developments occur.

## Contacts
- Incident Commander: [name]
- Security Lead: [name]
- Communications: [name]
```

### Post-Incident Report

```markdown
# Post-Incident Report

**Incident ID:** INC-2024-001
**Date:** 2024-01-15
**Duration:** 4 hours 23 minutes
**Severity:** SEV-1

## Executive Summary
Brief description of what happened and resolution.

## Timeline
| Time (UTC) | Event |
|------------|-------|
| 14:30 | Initial detection via Falco alert |
| 14:35 | Security team notified |
| 14:45 | Pod isolated |
| 15:00 | Root cause identified |
| 16:30 | Remediation complete |
| 18:53 | Incident closed |

## Root Cause Analysis
Detailed analysis of how the incident occurred.

## Impact Assessment
- Systems affected: 3 pods in production namespace
- Data exposure: No customer data compromised
- Service disruption: 15 minutes degraded performance

## Remediation Actions Taken
1. Compromised container isolated and terminated
2. Vulnerable image updated
3. Network policies strengthened

## Lessons Learned
- What worked well
- What could be improved
- Recommendations

## Action Items
| Item | Owner | Due Date | Status |
|------|-------|----------|--------|
| Update container scanning | Security | 2024-01-22 | Open |
| Implement additional monitoring | SRE | 2024-01-29 | Open |
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                INCIDENT RESPONSE BEST PRACTICES                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Preparation:                                                       │
│  ✓ Document playbooks for common scenarios                         │
│  ✓ Train team regularly with tabletop exercises                    │
│  ✓ Maintain up-to-date contact lists                               │
│  ✓ Test response procedures periodically                           │
│                                                                     │
│  Detection:                                                         │
│  ✓ Implement comprehensive monitoring                              │
│  ✓ Use automated detection tools (Falco, GuardDuty)                │
│  ✓ Set appropriate alert thresholds                                │
│  ✓ Minimize false positives                                        │
│                                                                     │
│  Response:                                                          │
│  ✓ Prioritize containment over investigation                       │
│  ✓ Preserve evidence before remediation                            │
│  ✓ Communicate clearly and regularly                               │
│  ✓ Document everything                                             │
│                                                                     │
│  Post-Incident:                                                     │
│  ✓ Conduct blameless post-mortems                                  │
│  ✓ Implement lessons learned                                       │
│  ✓ Update playbooks based on experience                            │
│  ✓ Share knowledge with the team                                   │
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
│  IR Phases:                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Preparation   │ Playbooks, training, tools                  │   │
│  │ Detection     │ Monitoring, alerting, analysis              │   │
│  │ Containment   │ Isolate, preserve evidence                  │   │
│  │ Eradication   │ Remove threat, patch vulnerabilities        │   │
│  │ Recovery      │ Restore services, verify clean              │   │
│  │ Post-Incident │ Document, learn, improve                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Tools:                                                         │
│  • Falco - Runtime threat detection                                │
│  • GuardDuty - AWS threat detection                                │
│  • Network policies - Pod isolation                                │
│  • Forensics toolkit - Evidence collection                         │
│                                                                     │
│  Success Factors:                                                   │
│  • Well-documented playbooks                                       │
│  • Regular training and drills                                     │
│  • Automated detection and response                                │
│  • Clear communication channels                                    │
│  • Blameless post-mortems                                          │
│                                                                     │
│  Next: Learn about DevSecOps best practices                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
