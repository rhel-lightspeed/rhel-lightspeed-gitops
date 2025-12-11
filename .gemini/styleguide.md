# GitOps Style Guide for ArgoCD/Kubernetes/OpenShift

This style guide defines standards for managing Kubernetes manifests and ArgoCD Applications in the rhel-lightspeed-gitops repository. All code reviews should enforce these conventions.

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

**Health Probes (MANDATORY)**

All Deployments MUST include BOTH liveness and readiness probes:

```yaml
# CORRECT
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

Guidelines:
- CPU requests: 100m minimum for standard apps, 2+ cores for GPU workloads
- Memory requests: 128Mi minimum, scaled appropriately for workload
- Limits should be 2-5x requests for burstable workloads
- GPU workloads: `nvidia.com/gpu` in both requests and limits

**Label Consistency**

- All resources MUST have `app: <name>` label
- Deployment selector MUST match pod template labels exactly
- Use `managed-by: argocd` label for GitOps tracking

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

### Resource Optimization

**Right-Sizing Guidelines**
- Standard web apps: 100m-500m CPU, 128Mi-512Mi memory
- API services: 500m-2 CPU, 512Mi-2Gi memory
- GPU workloads: 2+ CPU, 8Gi+ memory, specific GPU tolerations
- Batch jobs: Request minimum, limit generously

**Startup Performance**
- Use `initialDelaySeconds` appropriate for app startup time
- Set `readinessProbe.failureThreshold` high enough for slow starts
- Consider `startupProbe` for apps with very long initialization

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

## Security

### TLS and Network Security

**OpenShift Routes - TLS MANDATORY**

All Routes MUST include TLS configuration with edge termination:

```yaml
# CORRECT
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app
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

### Container Security

**Image Sources**
- Prefer Red Hat registry: `registry.redhat.io`
- Use official images from trusted sources
- Avoid `latest` tag (use digests)
- Review base image CVEs before introducing new images

**Security Contexts** (when applicable)
- Set `runAsNonRoot: true` for non-privileged apps
- Define `seccompProfile` for additional hardening
- Use `readOnlyRootFilesystem: true` when possible
- GPU workloads may require relaxed security contexts

### Secrets and Configuration

**Environment Variables**
- Never hardcode secrets in manifests (use Secrets or ExternalSecrets)
- Document sensitive environment variables with comments
- Use ConfigMaps for non-sensitive configuration

```yaml
# CORRECT
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: my-app-secrets
      key: api-key
```

## Miscellaneous

### OpenShift-Specific Patterns

**Routes vs Ingress**
- Always use OpenShift Routes (not Ingress resources)
- Routes provide integrated TLS certificate management
- Use `route.openshift.io/v1` API version

**Project/Namespace**
- Standard namespace: `rhel-lightspeed-ai`
- Operators may use `openshift-*` namespaces
- Never create resources in `default` namespace

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

### Persistent Storage

**PersistentVolumeClaims**
- Use descriptive names: `my-app-cache`, `my-app-config`
- Specify `storageClassName` explicitly
- Document access mode reasoning
- Size appropriately (model caches can be 50Gi+)

### ArgoCD Best Practices

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

## Common Mistakes to Flag

### Critical Issues (Block PR)
- ArgoCD Application without `destination.namespace`
- Container image using `:latest` or undigested tag
- Deployment without resource requests/limits
- Deployment without liveness/readiness probes
- OpenShift Route without TLS configuration
- Pod resource with `replicas` field (use Deployment)
- Secrets or credentials committed to repository

### High Priority Issues (Request changes)
- Inconsistent label selectors (selector doesn't match pod labels)
- Missing namespace specification in resources
- ArgoCD sync policy missing `selfHeal` or `allowEmpty`
- GPU workload without tolerations
- Missing `managed-by: argocd` label
- Probe `initialDelaySeconds` too short for app startup
- Resource limits without requests (or vice versa)

### Medium Priority Issues (Suggest improvements)
- Missing comments on complex configurations
- Resource requests/limits could be optimized
- Probe failure thresholds too aggressive
- No kustomization.yaml in application directory
- Inconsistent naming conventions
- Missing sync wave annotations for dependent resources
- Deployment strategy not specified (defaults to RollingUpdate)

### Low Priority Issues (Optional suggestions)
- Consider adding more descriptive labels
- Could benefit from additional documentation
- Port naming would improve readability
- Consider using ConfigMap for environment variables

## Review Process

When reviewing pull requests:

1. **Verify Correctness**: Check resource types, fields, and ArgoCD specs
2. **Validate Security**: Ensure TLS on Routes, digested images, no secrets
3. **Assess Resource Management**: Confirm requests/limits and probes exist
4. **Check Consistency**: Verify naming, labels, and organizational patterns
5. **Evaluate Maintainability**: Look for comments, clear structure, proper organization

Focus on correctness and security first, then efficiency and maintainability.
