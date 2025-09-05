# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a GitOps repository for managing applications deployed by ArgoCD in RHEL Lightspeed's OpenShift cluster. The repository contains Kubernetes manifests and ArgoCD Application resources that define deployments to be managed through GitOps.

## Repository Structure

- `gitops-applications/` - Contains ArgoCD Application manifests (*.application.yaml files) that define which applications ArgoCD should deploy and manage
- Individual application directories (e.g., `speedtest/`) - Contains Kubernetes manifests for specific applications:
  - `deployment.yaml` - Kubernetes deployment configuration
  - `service.yaml` - Service definitions to expose applications
  - `route.yaml` - OpenShift routes for external access
  - `kustomization.yaml` - Kustomize configuration for resource management

## Key Configuration Details

### Default Namespace
All applications are deployed to the `rhel-lightspeed-ai` namespace unless otherwise specified.

### ArgoCD Application Pattern
ArgoCD applications follow this structure:
- Metadata includes the application name and namespace (`rhel-lightspeed-ai`)
- Destination is always `in-cluster` targeting the `rhel-lightspeed-ai` namespace
- Source points to either:
  - External repositories (with namespace overrides via `kustomize.namespace`)
  - Local directories within this repository
- Sync policies typically include:
  - `automated.selfHeal: true` - Auto-remediate drift
  - `automated.prune: true/false` - Whether to delete resources not in git
  - `syncOptions.CreateNamespace=true` - Auto-create namespace if needed

### OpenShift-Specific Resources
This cluster uses OpenShift, so applications include:
- OpenShift Routes (apiVersion: route.openshift.io/v1) for external access
- TLS edge termination on routes with redirect policy

## Git Configuration

- Remote repository: `git@github.com:rhel-lightspeed/rhel-lightspeed-gitops.git`
- Main branch: `main`
- Renovate is configured for automated dependency updates with automerge

## Common Tasks

### Adding a New Application
1. Create a directory for the application with Kubernetes manifests
2. Include a `kustomization.yaml` to manage resources
3. Create an ArgoCD Application manifest in `gitops-applications/`
4. Ensure all resources specify `namespace: rhel-lightspeed-ai`
5. For external repositories, use `kustomize.namespace` to override namespaces

### Resource Requirements
Always include resource requests and limits in deployments:
- requests: Minimum guaranteed resources
- limits: Maximum allowed resources
- Include liveness and readiness probes for reliability