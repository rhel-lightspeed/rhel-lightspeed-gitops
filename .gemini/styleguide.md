# GitOps Style Guide for ArgoCD/Kubernetes/OpenShift

This style guide defines standards for managing Kubernetes manifests and ArgoCD Applications in the rhel-lightspeed-gitops repository. All code reviews should enforce these conventions.

## Table of Contents

1. [Quick Reference Checklist](#quick-reference-checklist)
2. [Correctness](#correctness)
   - [ArgoCD Application Manifests](#argocd-application-manifests)
   - [Kubernetes Resources](#kubernetes-resources)
3. [Health and Reliability](#health-and-reliability)
   - [Health Probes](#health-probes)
   - [Pod Disruption Budgets](#pod-disruption-budgets)
4. [Resource Management](#resource-management)
   - [Resource Requests and Limits](#resource-requests-and-limits)
   - [Horizontal Pod Autoscaling](#horizontal-pod-autoscaling)
5. [Security](#security)
   - [Pod Security Standards](#pod-security-standards)
   - [TLS and Network Security](#tls-and-network-security)
   - [Network Policies](#network-policies)
   - [Secrets Management](#secrets-management)
   - [RBAC and Service Accounts](#rbac-and-service-accounts)
   - [Container Security](#container-security)
6. [Efficiency](#efficiency)
   - [Container Image Management](#container-image-management)
   - [Resource Optimization](#resource-optimization)
7. [Maintainability](#maintainability)
   - [Naming Conventions](#naming-conventions)
   - [Directory Structure](#directory-structure)
   - [Documentation](#documentation)
8. [Observability](#observability)
   - [Metrics Exposure](#metrics-exposure)
   - [Logging Standards](#logging-standards)
9. [Specialized Workloads](#specialized-workloads)
   - [GPU Workloads](#gpu-workloads)
   - [Persistent Storage](#persistent-storage)
10. [OpenShift-Specific Patterns](#openshift-specific-patterns)
11. [ArgoCD Best Practices](#argocd-best-practices)
12. [Operational Procedures](#operational-procedures)
    - [Rollback Procedures](#rollback-procedures)
    - [Troubleshooting Guide](#troubleshooting-guide)
13. [Common Mistakes to Flag](#common-mistakes-to-flag)
14. [Review Process](#review-process)

---

## Quick Reference Checklist

Before submitting a PR, verify these requirements:

### Mandatory Fields

- [ ] `metadata.namespace: rhel-lightspeed-ai`
- [ ] `spec.replicas` specified (Deployments)
- [ ] `resources.requests` (cpu, memory)
- [ ] `resources.limits` (cpu, memory)
- [ ] `livenessProbe` configured
- [ ] `readinessProbe` configured
- [ ] Image uses SHA256 digest (not tag)
- [ ] Labels include `app: <name>`

### Security

- [ ] Route has TLS configuration
- [ ] No secrets in manifests
- [ ] Security context specified (non-GPU workloads)
- [ ] Running as non-root (when possible)

### ArgoCD

- [ ] `metadata.namespace: rhel-lightspeed-ai`
- [ ] `destination.namespace: rhel-lightspeed-ai`
- [ ] `syncPolicy.automated.selfHeal: true`
- [ ] `syncPolicy.automated.allowEmpty: false`
- [ ] `syncPolicy.automated.prune` explicitly set

### Documentation

- [ ] Comments explain non-obvious config
- [ ] Resource limits reasoning documented
- [ ] `kustomization.yaml` present

---

## Correctness

### ArgoCD Application Manifests

**Namespace Specifications (CRITICAL)**

Every Application MUST explicitly specify both:
- `metadata.namespace: rhel-lightspeed-ai`
- `spec.destination.namespace: rhel-lightspeed-ai`

Never omit destination namespace, even if targeting default namespace.

```yaml
# CORRECT
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: rhel-lightspeed-ai
spec:
  destination:
    name: in-cluster
    namespace: rhel-lightspeed-ai
  project: default
```

```yaml
# INCORRECT - Missing destination.namespace
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  destination:
    name: in-cluster
  project: default
```

**Sync Policy Standards**

Always include a complete sync policy:
- `syncPolicy.automated.selfHeal: true` for GitOps enforcement
- `syncPolicy.automated.allowEmpty: false` to prevent accidental empty syncs
- `syncPolicy.automated.prune` explicitly set (true/false) based on app requirements
- `syncOptions.CreateNamespace=true` for new namespaces

```yaml
# CORRECT - Complete sync policy
syncPolicy:
  automated:
    selfHeal: true
    prune: true
    allowEmpty: false
  syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground  # Wait for cascading deletes
    - PruneLast=true  # Prune after successful sync
```

Use `prune: true` for apps fully managed in this repo. Use `prune: false` for operators or apps with external resources.

**Source Configuration**

- Use `repoURL` with full HTTPS URLs (no SSH for public repos)
- Specify `targetRevision: main` explicitly
- For external repos, use `kustomize.namespace` to override namespace

### Kubernetes Resources

**Resource Type Validation**

- Pod resources MUST NOT have `spec.replicas` field (use Deployment instead)
- Deployment resources MUST have `spec.replicas` field
- StatefulSet resources MUST have `spec.serviceName` field

**Mandatory Fields for Deployments**

Every Deployment MUST include:
1. `spec.replicas` (set to 0 for paused apps)
2. `spec.selector.matchLabels` matching `spec.template.metadata.labels`
3. `metadata.namespace: rhel-lightspeed-ai`
4. `metadata.labels` with at minimum `app: <name>` label

**Label Consistency**

- All resources MUST have `app: <name>` label
- Deployment selector MUST match pod template labels exactly
- Use `managed-by: argocd` label for GitOps tracking

---

## Health and Reliability

### Health Probes

**Health Probes (MANDATORY)**

All Deployments MUST include BOTH liveness and readiness probes:

```yaml
# CORRECT - Standard probes for fast-starting apps
containers:
- name: my-app
  livenessProbe:
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 60
    periodSeconds: 30
    timeoutSeconds: 10
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
```

Common probe paths: `/health`, `/healthz`, `/ready`, `/readyz`, `/livez`

**Startup Probes for Slow-Starting Applications**

For applications with long initialization (AI models, large data loads), use startup probes to prevent liveness probes from killing pods during startup:

```yaml
# CORRECT - Startup probe for model loading (AI/ML workloads)
startupProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 60  # Allow 10 minutes for startup
  timeoutSeconds: 5

livenessProbe:
  httpGet:
    path: /health
    port: 8000
  periodSeconds: 30
  timeoutSeconds: 10
  failureThreshold: 3  # Can be aggressive after startup completes

readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
```

**Why startup probes matter**: Once the startup probe succeeds, it's disabled and liveness/readiness probes take over. This allows aggressive liveness checks in steady state while being patient during initialization.

### Pod Disruption Budgets

**Pod Disruption Budgets (REQUIRED for multi-replica apps)**

Prevent cluster maintenance from causing outages:

```yaml
# CORRECT - PDB for high availability
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app
  namespace: rhel-lightspeed-ai
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: my-app
```

**When to Use**:
- All applications with `replicas >= 2`
- Critical services that need guaranteed availability
- Use `minAvailable: 1` for 2-3 replicas
- Use `minAvailable: 2` or `maxUnavailable: 1` for 4+ replicas

**Exception**: Single-replica deployments don't need PDBs.

---

## Resource Management

### Resource Requests and Limits

**Resource Requests and Limits (MANDATORY)**

All containers MUST specify both requests and limits:

```yaml
# CORRECT
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

```yaml
# INCORRECT - Missing requests or limits
resources:
  limits:
    memory: "256Mi"
```

**Right-Sizing Guidelines**:
- Standard web apps: 100m-500m CPU, 128Mi-512Mi memory
- API services: 500m-2 CPU, 512Mi-2Gi memory
- GPU workloads: 2+ CPU, 8Gi+ memory, specific GPU tolerations
- Batch jobs: Request minimum, limit generously
- Limits should be 2-5x requests for burstable workloads

### Horizontal Pod Autoscaling

**When to Use HPA**
- Stateless services with variable load
- API services that can scale horizontally
- NOT for GPU workloads (expensive, slow startup)
- NOT for StatefulSets without careful configuration

```yaml
# CORRECT - HPA for stateless service
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
  namespace: rhel-lightspeed-ai
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Prevent flapping
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

**Best Practices**:
- Set `minReplicas: 2` minimum for production availability
- Use stabilization windows to prevent rapid scaling oscillation
- Monitor HPA metrics to tune thresholds

---

## Security

### Pod Security Standards

**Security Context Requirements (MANDATORY for non-GPU workloads)**

All non-GPU workloads MUST specify security contexts:

```yaml
# CORRECT - Restricted security context
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: my-app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
          - ALL
```

**OpenShift SCCs (Security Context Constraints)**
- Default to `restricted-v2` SCC
- Document when `anyuid` or custom SCCs are required
- GPU workloads typically need `nvidia-gpu` SCC
- Never use `privileged` without security review

**tmpfs for Read-Only Root Filesystems**:

When using `readOnlyRootFilesystem: true`, mount writable directories:

```yaml
volumeMounts:
- name: tmp
  mountPath: /tmp
- name: cache
  mountPath: /.cache

volumes:
- name: tmp
  emptyDir: {}
- name: cache
  emptyDir: {}
```

### TLS and Network Security

**OpenShift Routes - TLS MANDATORY**

All Routes MUST include TLS configuration with edge termination:

```yaml
# CORRECT
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app
  namespace: rhel-lightspeed-ai
spec:
  to:
    kind: Service
    name: my-app
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

```yaml
# INCORRECT - Missing TLS configuration
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app
spec:
  to:
    kind: Service
    name: my-app
```

**Port Naming**
- Always name ports in Services and Deployments: `name: http`, `name: https`
- Routes should reference named ports: `targetPort: http`

### Network Policies

**NetworkPolicy (RECOMMENDED for production)**

Implement zero-trust networking with default-deny policies:

```yaml
# Deny all ingress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-app-deny-all
  namespace: rhel-lightspeed-ai
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  - Egress
---
# Allow specific ingress from OpenShift router
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-app-allow-ingress
  namespace: rhel-lightspeed-ai
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress
    ports:
    - protocol: TCP
      port: 8080
---
# Allow egress for DNS and external HTTPS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-app-allow-egress
  namespace: rhel-lightspeed-ai
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Egress
  egress:
  - to:  # DNS
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-dns
    ports:
    - protocol: UDP
      port: 53
  - to:  # External HTTPS (APIs, registries)
    ports:
    - protocol: TCP
      port: 443
```

**Best Practices**:
- Start with deny-all, then whitelist necessary traffic
- AI workloads often need egress to model registries (HuggingFace, etc.)
- Test NetworkPolicies in non-production first
- Document egress requirements for each application

### Secrets Management

**External Secrets Operator (RECOMMENDED)**

Never store secrets in Git. Use External Secrets Operator:

```yaml
# CORRECT - External Secret referencing vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: rhel-lightspeed-ai
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: my-app-secrets
    creationPolicy: Owner
  data:
  - secretKey: api-key
    remoteRef:
      key: secret/data/my-app
      property: api-key
```

**Sealed Secrets (Alternative for GitOps-committed secrets)**

For encrypted secrets that can be committed to Git:

```bash
# Generate sealed secret
kubeseal --format=yaml < secret.yaml > sealed-secret.yaml
```

**Environment Variable References**:

```yaml
# CORRECT - Reference secrets, never hardcode
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: my-app-secrets
      key: api-key
```

**ConfigMaps vs Secrets**:
- ConfigMaps: Non-sensitive configuration, URLs, feature flags
- Secrets: API keys, tokens, passwords, certificates

### RBAC and Service Accounts

**Service Accounts (RECOMMENDED for apps accessing K8s API)**

Create dedicated service accounts with minimal permissions:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: rhel-lightspeed-ai
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app
  namespace: rhel-lightspeed-ai
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app
  namespace: rhel-lightspeed-ai
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: rhel-lightspeed-ai
roleRef:
  kind: Role
  name: my-app
  apiGroup: rbac.authorization.k8s.io
```

**Reference in Deployment**:

```yaml
spec:
  template:
    spec:
      serviceAccountName: my-app
      automountServiceAccountToken: false  # If K8s API access not needed
```

**Best Practices**:
- One ServiceAccount per application
- Disable token auto-mount if K8s API access not needed
- Use least-privilege RBAC rules
- Document why specific permissions are required

### Container Security

**Image Sources**
- Prefer Red Hat registry: `registry.redhat.io`
- Use official images from trusted sources
- Avoid `latest` tag (use digests)
- Review base image CVEs before introducing new images

---

## Efficiency

### Container Image Management

**Image Digest Pinning (REQUIRED)**

All container images MUST be pinned to SHA256 digests, not tags:

```yaml
# CORRECT - Digest pinned
image: docker.io/library/nginx@sha256:abcd1234...

# INCORRECT - Tag-based (even with version)
image: docker.io/library/nginx:1.25
image: ghcr.io/librespeed/speedtest:latest
```

Tag-based images can change, breaking reproducibility and security. Renovate handles digest updates automatically.

**Exceptions**: Images in active development may temporarily use tags, but MUST be converted to digests before merging to main branch.

**Automated Digest Updates with Renovate**

This repository uses Renovate to automatically update image digests. Renovate creates PRs for digest updates which auto-merge if checks pass.

**Manual Digest Conversion**:

```bash
# Convert tag to digest using skopeo
skopeo inspect docker://docker.io/vllm/vllm-openai:v0.10.2 | jq -r '.Digest'
# Returns: sha256:abc123...

# Or using crane
crane digest docker.io/vllm/vllm-openai:v0.10.2
```

**Image Pull Configuration**:

```yaml
spec:
  containers:
  - name: my-app
    image: registry.redhat.io/my-app@sha256:abc...
    imagePullPolicy: IfNotPresent  # Don't re-pull digested images
  imagePullSecrets:
  - name: registry-credentials  # For private registries
```

### Resource Optimization

**Startup Performance**
- Use `initialDelaySeconds` appropriate for app startup time
- Set `readinessProbe.failureThreshold` high enough for slow starts
- Use `startupProbe` for apps with very long initialization (see Health Probes section)

---

## Maintainability

### Naming Conventions

**Resource Names**
- Use lowercase kebab-case: `my-app`, `rhelaiis`, `cuda-vectoradd`
- Keep names short but descriptive
- Avoid redundant suffixes (`-deployment`, `-svc`) - type is in `kind` field
- Match ArgoCD Application name with directory name

**File Naming**
- Use descriptive names: `deployment.yaml`, `service.yaml`, `route.yaml`
- One resource type per file
- Use `*.application.yaml` for ArgoCD Applications in `gitops-applications/`

**Label Values**
- Match primary resource name
- Use consistent labeling: `app: my-app`, `managed-by: argocd`
- Avoid changing labels on existing resources (causes recreation)

### Directory Structure

**Application Organization**
```
/
├── gitops-applications/
│   └── my-app.application.yaml
├── my-app/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── route.yaml
```

**Kustomization Files**

Every application directory MUST include `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: rhel-lightspeed-ai

resources:
  - deployment.yaml
  - service.yaml
  - route.yaml

commonLabels:
  app: my-app
  managed-by: argocd
```

### Documentation

**YAML Comments**
- Add comments for non-obvious configurations
- Document why specific values were chosen (especially resource limits)
- Explain GPU tolerations and node affinity rules
- Note any temporary workarounds

```yaml
# Using Recreate strategy due to PVC access mode (ReadWriteOnce)
strategy:
  type: Recreate

# High failure threshold for slow model loading (10+ minutes)
readinessProbe:
  failureThreshold: 84
```

---

## Observability

### Metrics Exposure

**Metrics Exposure (REQUIRED for production apps)**

Expose Prometheus metrics for monitoring:

```yaml
# Service with metrics annotations
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: rhel-lightspeed-ai
  labels:
    app: my-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: my-app
```

**ServiceMonitor for Prometheus Operator**:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: rhel-lightspeed-ai
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: http
    interval: 30s
    path: /metrics
```

**Standard Labels for Observability**:

```yaml
metadata:
  labels:
    app: my-app
    version: v1.2.3
    component: api
    part-of: my-service
    managed-by: argocd
```

### Logging Standards

**Best Practices**:
- Log to stdout/stderr (captured by OpenShift logging)
- Use structured logging (JSON format preferred)
- Include correlation IDs for request tracing
- Set appropriate log levels via environment variables

```yaml
env:
- name: LOG_LEVEL
  value: "info"
- name: LOG_FORMAT
  value: "json"
```

---

## Specialized Workloads

### GPU Workloads

**GPU Node Tolerations**

GPU workloads MUST include appropriate tolerations:

```yaml
tolerations:
- key: "nvidia.com/gpu"
  operator: "Equal"
  value: "A10G"  # Or appropriate GPU type
  effect: "NoSchedule"
```

**GPU Resource Specifications**
- Request and limit must match: `nvidia.com/gpu: 1`
- Set `CUDA_VISIBLE_DEVICES` environment variable explicitly
- Include large `/dev/shm` for CUDA operations:

```yaml
volumes:
- name: dshm
  emptyDir:
    medium: Memory
    sizeLimit: 16Gi
```

**GPU Workload Resource Requirements**
- Minimum 2 CPU cores, 8Gi memory per GPU
- Use `strategy.type: Recreate` for PVC-backed deployments
- Set high `failureThreshold` for readiness probes (model loading is slow)
- Use startup probes for very long initialization

**Security Context Exception**: GPU workloads may require relaxed security contexts due to NVIDIA driver requirements. Document the specific requirements.

### Persistent Storage

**PersistentVolumeClaims**
- Use descriptive names: `my-app-cache`, `my-app-config`
- Specify `storageClassName` explicitly
- Document access mode reasoning
- Size appropriately (model caches can be 50Gi+)

---

## OpenShift-Specific Patterns

**Routes vs Ingress**
- Always use OpenShift Routes (not Ingress resources)
- Routes provide integrated TLS certificate management
- Use `route.openshift.io/v1` API version

**Project/Namespace**
- Standard namespace: `rhel-lightspeed-ai`
- Operators may use `openshift-*` namespaces
- Never create resources in `default` namespace

---

## ArgoCD Best Practices

**Application Organization**
- One Application per deployable unit
- Group related resources in same directory
- Use Application Sets for multi-environment patterns
- Leverage Kustomize for environment-specific overrides

**Sync Waves** (when needed)

Use sync waves for ordered deployment:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

Common waves:
- 0: Namespaces, CustomResourceDefinitions
- 1: PersistentVolumeClaims, ConfigMaps, Secrets
- 2: Deployments, StatefulSets
- 3: Services, Routes

---

## Operational Procedures

### Rollback Procedures

ArgoCD provides multiple rollback options:

**1. Git Revert (RECOMMENDED for audit trail)**

```bash
# Revert problematic commit
git revert abc123
git push

# ArgoCD auto-syncs the revert
```

**2. ArgoCD CLI Rollback**

```bash
# Rollback to previous revision
argocd app rollback my-app

# Rollback to specific revision
argocd app rollback my-app --revision 123
```

**3. ArgoCD UI Rollback**
- Navigate to Application -> History
- Select previous revision -> Rollback

**Application Health Status**:
- **Healthy**: All resources running correctly
- **Progressing**: Deployment in progress
- **Degraded**: Resources failing
- **Suspended**: Sync paused

### Troubleshooting Guide

**Pod Not Starting**

```
Pod in Pending state?
├── kubectl describe pod <name>
│   ├── "Insufficient cpu/memory" -> Increase node capacity or reduce requests
│   ├── "No nodes match tolerations" -> Check GPU tolerations
│   └── "PVC not bound" -> Check PVC status, storage class

Pod in CrashLoopBackOff?
├── kubectl logs <pod> --previous
│   ├── OOMKilled -> Increase memory limits
│   ├── Exit code 137 -> OOM or killed by K8s
│   └── Application error -> Fix application code

Pod in ImagePullBackOff?
├── kubectl describe pod <name>
│   ├── "manifest unknown" -> Verify digest exists
│   ├── "unauthorized" -> Check imagePullSecrets
│   └── "timeout" -> Network/registry issue
```

**ArgoCD Sync Failures**

```
Application stuck in "Progressing"?
├── Check sync status in ArgoCD UI
│   ├── Resource hooks pending -> Check hook job logs
│   ├── Health check failing -> Review resource health checks
│   └── Timeout -> Increase sync timeout

Application shows "OutOfSync"?
├── Check diff in ArgoCD UI
│   ├── Managed field conflicts -> Use ServerSideApply sync option
│   ├── Resource missing -> Check if manually deleted
│   └── Namespace mismatch -> Verify namespace overrides
```

---

## Common Mistakes to Flag

### Critical Issues (Block PR)
- ArgoCD Application without `destination.namespace`
- Container image using `:latest` or undigested tag
- Deployment without resource requests/limits
- Deployment without liveness/readiness probes
- OpenShift Route without TLS configuration
- Pod resource with `replicas` field (use Deployment)
- Secrets or credentials committed to repository
- Missing security context on non-GPU workloads

### High Priority Issues (Request changes)
- Inconsistent label selectors (selector doesn't match pod labels)
- Missing namespace specification in resources
- ArgoCD sync policy missing `selfHeal` or `allowEmpty`
- GPU workload without tolerations
- Missing `managed-by: argocd` label
- Probe `initialDelaySeconds` too short for app startup
- Resource limits without requests (or vice versa)
- Multi-replica deployment without PodDisruptionBudget

### Medium Priority Issues (Suggest improvements)
- Missing comments on complex configurations
- Resource requests/limits could be optimized
- Probe failure thresholds too aggressive
- No kustomization.yaml in application directory
- Inconsistent naming conventions
- Missing sync wave annotations for dependent resources
- Deployment strategy not specified (defaults to RollingUpdate)
- Missing startup probe for slow-starting applications
- No ServiceMonitor for production applications

### Low Priority Issues (Optional suggestions)
- Consider adding more descriptive labels
- Could benefit from additional documentation
- Port naming would improve readability
- Consider using ConfigMap for environment variables
- Consider adding NetworkPolicy for zero-trust security

---

## Review Process

When reviewing pull requests:

1. **Verify Correctness**: Check resource types, fields, and ArgoCD specs
2. **Validate Security**: Ensure TLS on Routes, digested images, no secrets, security contexts
3. **Assess Resource Management**: Confirm requests/limits and probes exist
4. **Check Reliability**: Verify PDBs for multi-replica apps, appropriate probe settings
5. **Check Consistency**: Verify naming, labels, and organizational patterns
6. **Evaluate Maintainability**: Look for comments, clear structure, proper organization
7. **Consider Observability**: Check for metrics exposure and logging configuration

Focus on correctness and security first, then reliability and efficiency, then maintainability.
