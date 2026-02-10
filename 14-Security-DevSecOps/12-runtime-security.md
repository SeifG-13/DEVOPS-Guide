# Runtime Security

## Runtime Security Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RUNTIME SECURITY                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What is Runtime Security?                                         │
│  • Protecting applications while they are running                  │
│  • Detecting and preventing threats in real-time                   │
│  • Monitoring behavior and anomalies                               │
│  • Responding to security incidents                                │
│                                                                     │
│  Runtime Security Layers:                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Application │ RASP, WAF, input validation                   │   │
│  │ Container   │ Syscall filtering, behavior monitoring        │   │
│  │ Kubernetes  │ Admission control, network policies           │   │
│  │ Host        │ SELinux, AppArmor, seccomp                    │   │
│  │ Network     │ Traffic monitoring, IDS/IPS                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Falco - Runtime Threat Detection

### Installation

```bash
# Helm installation
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true

# Verify installation
kubectl get pods -n falco
```

### Falco Rules

```yaml
# Custom Falco rules
# /etc/falco/rules.d/custom-rules.yaml

# Detect shell spawned in container
- rule: Shell Spawned in Container
  desc: Detect shell being spawned in a container
  condition: >
    spawned_process and
    container and
    shell_procs
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname
    cmdline=%proc.cmdline image=%container.image.repository)
  priority: WARNING
  tags: [container, shell, mitre_execution]

# Detect sensitive file access
- rule: Read Sensitive File in Container
  desc: Detect reading of sensitive files in containers
  condition: >
    open_read and
    container and
    sensitive_files
  output: >
    Sensitive file read in container
    (user=%user.name file=%fd.name container=%container.name
    image=%container.image.repository)
  priority: WARNING
  tags: [container, filesystem, mitre_credential_access]

# Detect package management in container
- rule: Package Management in Container
  desc: Package managers shouldn't run in production
  condition: >
    spawned_process and
    container and
    package_mgmt_procs
  output: >
    Package management process launched
    (user=%user.name command=%proc.cmdline
    container=%container.name image=%container.image.repository)
  priority: ERROR
  tags: [container, process, mitre_persistence]

# Detect crypto mining
- rule: Crypto Mining Detected
  desc: Detect cryptocurrency mining processes
  condition: >
    spawned_process and
    container and
    (proc.name in (crypto_miner_procs) or
     proc.cmdline contains "stratum+tcp" or
     proc.cmdline contains "cryptonight")
  output: >
    Crypto mining detected
    (user=%user.name command=%proc.cmdline
    container=%container.name)
  priority: CRITICAL
  tags: [container, mitre_resource_hijacking]

# Detect network tools
- rule: Network Reconnaissance in Container
  desc: Detect network scanning tools
  condition: >
    spawned_process and
    container and
    proc.name in (network_tool_procs)
  output: >
    Network tool launched in container
    (user=%user.name command=%proc.cmdline
    container=%container.name)
  priority: WARNING
  tags: [container, network, mitre_discovery]
```

### Falco Configuration

```yaml
# values.yaml for Helm
falco:
  json_output: true
  log_level: info

  rules_file:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/falco_rules.local.yaml
    - /etc/falco/rules.d

# Falcosidekick outputs
falcosidekick:
  enabled: true
  config:
    slack:
      webhookurl: "https://hooks.slack.com/services/XXX"
      minimumpriority: "warning"

    pagerduty:
      apikey: ""
      minimumpriority: "error"

    elasticsearch:
      hostport: "http://elasticsearch:9200"
      index: "falco"
      minimumpriority: "warning"

  webui:
    enabled: true
    service:
      type: ClusterIP
```

---

## Kubernetes Runtime Security

### Pod Security Admission

```yaml
# Namespace with Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Compliant Pod
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:latest
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      resources:
        limits:
          memory: "256Mi"
          cpu: "500m"
        requests:
          memory: "128Mi"
          cpu: "250m"
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

### Seccomp Profiles

```yaml
# Custom seccomp profile
# /var/lib/kubelet/seccomp/profiles/restricted.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "read", "write", "open", "close",
        "stat", "fstat", "lstat",
        "poll", "lseek", "mmap", "mprotect",
        "munmap", "brk", "rt_sigaction",
        "rt_sigprocmask", "ioctl", "access",
        "pipe", "select", "sched_yield",
        "mremap", "msync", "mincore",
        "madvise", "dup", "dup2", "nanosleep",
        "getpid", "socket", "connect", "accept",
        "sendto", "recvfrom", "bind", "listen",
        "getsockname", "getpeername", "setsockopt",
        "getsockopt", "clone", "fork", "execve",
        "exit", "wait4", "kill", "uname",
        "fcntl", "flock", "fsync", "fdatasync",
        "getcwd", "chdir", "fchdir", "readlink",
        "chmod", "fchmod", "chown", "fchown",
        "umask", "gettimeofday", "getrlimit",
        "getuid", "getgid", "setuid", "setgid",
        "geteuid", "getegid", "getgroups",
        "setgroups", "setpgid", "getppid",
        "setsid", "getpgid", "getsid",
        "capget", "capset", "rt_sigsuspend",
        "sigaltstack", "utime", "statfs", "fstatfs",
        "prctl", "arch_prctl", "futex",
        "set_tid_address", "exit_group",
        "epoll_create", "epoll_ctl", "epoll_wait",
        "set_robust_list", "get_robust_list",
        "epoll_create1", "eventfd2", "timerfd_create",
        "timerfd_settime", "timerfd_gettime",
        "accept4", "signalfd4", "eventfd", "pipe2",
        "inotify_init1", "pread64", "pwrite64",
        "readv", "writev", "preadv", "pwritev",
        "getrandom", "memfd_create"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
---
# Pod with custom seccomp
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/restricted.json
  containers:
    - name: app
      image: myapp:latest
```

### AppArmor Profiles

```yaml
# AppArmor profile
# /etc/apparmor.d/k8s-deny-write
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  # Allow reads
  file,

  # Deny writes to most places
  deny /bin/** w,
  deny /sbin/** w,
  deny /usr/** w,
  deny /etc/** w,

  # Allow writes to specific paths
  /tmp/** rw,
  /var/tmp/** rw,
}
---
# Pod with AppArmor
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-deny-write
spec:
  containers:
    - name: app
      image: myapp:latest
```

---

## Container Runtime Security

### gVisor (Sandbox Runtime)

```yaml
# RuntimeClass for gVisor
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
# Pod using gVisor
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-pod
spec:
  runtimeClassName: gvisor
  containers:
    - name: app
      image: myapp:latest
```

### Kata Containers

```yaml
# RuntimeClass for Kata
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata-qemu
overhead:
  podFixed:
    memory: "160Mi"
    cpu: "250m"
---
# Pod using Kata
apiVersion: v1
kind: Pod
metadata:
  name: kata-pod
spec:
  runtimeClassName: kata
  containers:
    - name: app
      image: myapp:latest
```

---

## Network Runtime Security

### Cilium Network Policies

```yaml
# Cilium Network Policy with L7 rules
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-rules
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/api/v1/users"
              - method: "POST"
                path: "/api/v1/users"
  egress:
    - toEndpoints:
        - matchLabels:
            app: database
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
    - toFQDNs:
        - matchName: "api.external.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

### Service Mesh Security (Istio)

```yaml
# Istio PeerAuthentication - mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
---
# Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/frontend"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/v1/*"]
    - from:
        - source:
            principals: ["cluster.local/ns/monitoring/sa/prometheus"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/metrics"]
```

---

## Application Runtime Protection

### Web Application Firewall (WAF)

```yaml
# ModSecurity with NGINX Ingress
apiVersion: v1
kind: ConfigMap
metadata:
  name: modsecurity-config
  namespace: ingress-nginx
data:
  modsecurity.conf: |
    SecRuleEngine On
    SecRequestBodyAccess On
    SecResponseBodyAccess Off
    SecRequestBodyLimit 13107200
    SecRequestBodyNoFilesLimit 131072
    SecAuditEngine RelevantOnly
    SecAuditLogRelevantStatus "^(?:5|4(?!04))"
    SecAuditLog /var/log/modsec_audit.log

    # OWASP CRS
    Include /etc/modsecurity.d/owasp-crs/crs-setup.conf
    Include /etc/modsecurity.d/owasp-crs/rules/*.conf
---
# Ingress with ModSecurity
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-modsecurity-crs: "true"
    nginx.ingress.kubernetes.io/modsecurity-snippet: |
      SecRuleEngine On
      SecRule ARGS "@contains <script>" "id:1,deny,status:403"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

### Rate Limiting

```yaml
# NGINX Ingress rate limiting
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
    nginx.ingress.kubernetes.io/limit-rpm: "100"
    nginx.ingress.kubernetes.io/limit-whitelist: "10.0.0.0/8"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

---

## Monitoring and Alerting

### Prometheus Alerts for Security

```yaml
# Security alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: security-alerts
  namespace: monitoring
spec:
  groups:
    - name: security
      rules:
        - alert: PrivilegedContainerRunning
          expr: |
            kube_pod_container_info{container!=""}
            * on(pod, namespace)
            kube_pod_container_security_context_privileged{privileged="true"} > 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Privileged container running"
            description: "Container {{ $labels.container }} in pod {{ $labels.pod }} is running as privileged"

        - alert: PodRunningAsRoot
          expr: |
            kube_pod_container_info{container!=""}
            * on(pod, namespace)
            kube_pod_container_security_context_runAsUser{uid="0"} > 0
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "Pod running as root"
            description: "Container {{ $labels.container }} in pod {{ $labels.pod }} is running as root"

        - alert: FalcoAlert
          expr: falco_events{priority="Critical"} > 0
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: "Falco critical alert"
            description: "Falco detected a critical security event"

        - alert: HighNetworkTraffic
          expr: |
            rate(container_network_receive_bytes_total[5m]) > 100000000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High network traffic detected"
            description: "Container {{ $labels.container }} receiving unusually high traffic"
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                 RUNTIME SECURITY BEST PRACTICES                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Container Security:                                                │
│  ✓ Run as non-root user                                            │
│  ✓ Use read-only root filesystem                                   │
│  ✓ Drop all capabilities                                           │
│  ✓ Enable seccomp profiles                                         │
│  ✓ Use AppArmor/SELinux                                            │
│                                                                     │
│  Kubernetes Security:                                               │
│  ✓ Enable Pod Security Standards (restricted)                      │
│  ✓ Implement network policies                                      │
│  ✓ Use runtime threat detection (Falco)                            │
│  ✓ Limit resource usage                                            │
│                                                                     │
│  Network Security:                                                  │
│  ✓ Enable mTLS between services                                    │
│  ✓ Implement authorization policies                                │
│  ✓ Use WAF for external traffic                                    │
│  ✓ Monitor for anomalies                                           │
│                                                                     │
│  Monitoring:                                                        │
│  ✓ Centralize security logs                                        │
│  ✓ Set up real-time alerting                                       │
│  ✓ Regular security audits                                         │
│  ✓ Incident response procedures                                    │
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
│  Runtime Security Tools:                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Falco         │ Real-time threat detection                  │   │
│  │ gVisor        │ Container sandboxing                        │   │
│  │ Kata          │ VM-based container isolation                │   │
│  │ Cilium        │ eBPF-based network security                 │   │
│  │ Istio         │ Service mesh security                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Security Controls:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Pod Security  │ Enforce security contexts                   │   │
│  │ Seccomp       │ Syscall filtering                           │   │
│  │ AppArmor      │ Application confinement                     │   │
│  │ NetworkPolicy │ Network segmentation                        │   │
│  │ mTLS          │ Service-to-service encryption               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Principles:                                                    │
│  • Defense in depth                                                │
│  • Least privilege                                                 │
│  • Continuous monitoring                                           │
│  • Automated response                                              │
│                                                                     │
│  Next: Learn about compliance and auditing                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
