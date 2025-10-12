# CRUSH - Kubernetes Infrastructure with Flux

## Commands

### Validate Kubernetes manifests
```bash
kubectl kustomize prod/ --enable-helm
kubectl kustomize yuzu/ --enable-helm
kubectl kustomize common/cert-manager
```

### Check for YAML errors
```bash
yamllint prod/ yuzu/ common/
```

## Code Style Guidelines

### File Organization
- Use directory-based organization: `<environment>/<service>/` (e.g., `prod/traefik/`, `yuzu/vaultwarden/`)
- Common/shared configs go in `common/<service>/`
- Split Helm installations into `init/`, `release/`, `secrets/`, `config/` subdirectories when needed

### Kubernetes Manifests
- Always specify `apiVersion`, `kind`, `metadata.name`, `metadata.namespace`
- Use `app.kubernetes.io/name` and `app.kubernetes.io/instance` labels consistently
- Pin versions explicitly in HelmRelease specs (e.g., `version: 'v1.18.2'`)
- Set `interval: 5m` for HelmRelease specs
- Use `crds: CreateReplace` for both install and upgrade in HelmRelease

### Secrets Management
- Use ExternalSecret resources from external-secrets.io/v1
- Reference ClusterSecretStore named `production`
- Set `refreshInterval: "1h"` and `refreshPolicy: Periodic`
- Use `creationPolicy: Owner` and `deletionPolicy: Delete`

### Kustomization Files
- List resources in logical order: flux-system first, then dependencies, then applications
- Comment out disabled resources with `#` instead of removing them

### Resource Specifications
- Always specify both `requests` and `limits` for CPU and memory
- Use consistent format: `cpu: 250m`, `memory: 512Mi`
