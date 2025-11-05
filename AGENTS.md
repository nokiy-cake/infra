# Repository Guidelines

## Project Structure & Module Organization
- `common/` holds shared building blocks (cert-manager, cilium, traefik, etc.), each split into `init/`, `release/`, `config/`, and `secrets/` with a local `kustomization.yaml`.
- Environment overlays live in `prod/`, `yuzu/`, and `unit/`. They re-use common components and add environment-scoped Kustomizations (for example `prod/flux-system/` for the production Flux bootstrap).
- `unit/` provides the lightest manifest set for smoke validation, while `yuzu/` is the staging mirror of production. Keep new components disabled by default and opt-in per environment.

## Build, Test, and Development Commands
- `kubectl kustomize <env>/ --enable-helm` renders manifests for an environment (`prod/`, `yuzu/`, or `unit/`) and should stay warning-free.
- `flux diff kustomization <name> --path <dir>` (e.g. `flux diff kustomization prod --path prod/`) previews the changes Flux will apply before opening a PR.
- `yamllint common/ prod/ yuzu/ unit/` catches indentation and formatting regressions early.
- After merging, trigger reconciliation with `flux reconcile kustomization <name> --with-source` if an urgent rollout is needed.

## Coding Style & Naming Conventions
- Author YAML with two-space indentation, sorted keys, and comments only where they clarify intent.
- Always apply the Kubernetes recommended labels (`app.kubernetes.io/name`, `instance`, `component`, `part-of`) so Flux diffs stay readable.
- File names follow `kebab-case.yaml`; secrets and Helm releases live under the dedicated subfolders described above.
- Keep HelmRelease values minimal and override via environment-specific patches instead of editing shared defaults.

## Testing Guidelines
- Validate that `kubectl kustomize unit/ --enable-helm` renders cleanly before requesting review.
- For new services, add them to `unit/` in dry-run mode first; promote to `yuzu/` or `prod/` only after verification.
- Prefer `kubectl diff --server-side -f <rendered.yaml>` against the target cluster during manual validation.
- Ensure ExternalSecret references align with the correct 1Password item name; missing keys break Flux sync.

## Commit & Pull Request Guidelines
- Follow the existing Git history: short, imperative subjects such as “Reduce CPU limit for forgejo-runner” describe both action and scope. Avoid ambiguous subjects like “update”.
- Each PR should include: purpose, affected environments, verification commands run, and any manual steps required after deploy.
- Attach relevant diffs (`flux diff` output or `kubectl kustomize` snippets) and mention any secrets or credentials that need rotation.
- Link to the tracked issue or change request when available, and request a second review for production-impacting updates.

## Security & Configuration Tips
- Never commit raw secrets. Use External Secrets Operator definitions under `secrets/` and confirm the referenced `ClusterSecretStore` is present.
- Keep CRDs versioned in their owning component (e.g. `prod/kubeblocks/crds/`) and update them in lockstep with Helm chart bumps.

---

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **GitOps-based Kubernetes Infrastructure Repository** using **Flux CD v2** to manage multiple Kubernetes clusters. The repository implements Infrastructure as Code (IaC) principles with Kubernetes manifests, Helm charts, and Kustomize for declarative cluster management.

- **Primary Tool:** Flux CD v2 (GitOps continuous deployment)
- **Configuration:** YAML/Kubernetes manifests (~242 files, ~179k lines)
- **Environments:** prod (production), yuzu (staging/dev), unit (test)
- **Secrets:** 1Password integration via External Secrets Operator
- **Dependency Updates:** Renovate Bot automation

## Repository Structure

```
infra/
├── common/              # Shared infrastructure components (17 services)
├── prod/                # Production environment configs
├── yuzu/                # Development/staging environment configs
├── unit/                # Test cluster configs
├── CRUSH.md             # Development guidelines
├── renovate.json        # Dependency update automation
└── README.md            # Project README
```

### Common Components

Shared services deployed across environments in `common/`:

- **cert-manager/** - TLS certificate management (Let's Encrypt)
- **cilium/** - CNI and network policies
- **cnpg/** - CloudNative PostgreSQL operator
- **external-secrets/** - 1Password secret injection
- **forgejo/** - Git forge repository platform
- **forgejo-runner/** - CI/CD runner
- **hetzner/** - Hetzner cloud provider integration
- **k3s/** - Lightweight Kubernetes distribution
- **nokiy/** - Custom application (cms/web/postgres/redis)
- **redis-operator/** - Redis HA operator
- **simple-backup/** - Backup management
- **synapse/** - Matrix homeserver
- **tailscale/** - VPN and secure networking
- **traefik/** - Ingress controller and reverse proxy
- **velero/** - Disaster recovery and backup

Each component follows the pattern:
```
common/<service>/
├── init/          # Initialization (namespace, helm-repository)
├── release/       # HelmRelease definitions
├── config/        # ConfigMaps and configurations
├── secrets/       # ExternalSecret definitions
└── kustomization.yaml
```

## Development Commands

### Validation & Linting

```bash
# Validate all Kubernetes manifests with Helm
kubectl kustomize prod/ --enable-helm
kubectl kustomize yuzu/ --enable-helm
kubectl kustomize common/cert-manager --enable-helm

# Lint YAML files
yamllint prod/ yuzu/ common/
```

### Viewing Environment Configurations

```bash
# Preview production manifests
kubectl kustomize prod/ --enable-helm > prod-manifests.yaml

# Preview staging manifests
kubectl kustomize yuzu/ --enable-helm > yuzu-manifests.yaml

# Preview specific component
kubectl kustomize common/traefik --enable-helm
```

### Testing Environments

- **unit/** - Minimal configuration for validation
- **yuzu/** - Full feature set for pre-production testing
- **prod/** - Production workloads (subset of yuzu services)

## Architecture Overview

### Multi-Environment Strategy

**Shared Components** (`common/`) → **Environment Overrides** (`prod/`, `yuzu/`) → **Cluster Deployment**

- Common configurations define base functionality
- Environments specify which services to enable/disable
- Kustomize patches apply environment-specific settings
- Flux CD continuously reconciles state with Git

### Dependency Chain

```
flux-system (bootstrap)
  ├── k3s
  ├── cilium (requires k3s)
  ├── traefik (requires cilium)
  ├── cert-manager
  ├── external-secrets
  ├── tailscale
  ├── cnpg, redis-operator
  └── Applications (require all above)
```

### Network Architecture

- **Primary Ingress:** Traefik with multiple entrypoints:
  - `web` - HTTP (port 80)
  - `websecure` - HTTPS (port 443)
  - `webinternal` - HTTP internal
  - `websecureinternal` - HTTPS internal
- **TLS:** cert-manager with Let's Encrypt integration
- **VPN:** Tailscale for secure network access
- **CNI:** Cilium for network policies

### Secrets Management

- **Backend:** 1Password
- **Integration:** External Secrets Operator
- **Pattern:** ExternalSecret resources with `ClusterSecretStore: production`
- **Refresh:** 1-hour intervals via periodic policy
- **Lifecycle:** Owner creation, Delete on removal

Example ExternalSecret:
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: my-secret
  namespace: my-namespace
spec:
  refreshInterval: 1h
  refreshPolicy: Periodic
  secretStoreRef:
    name: production
    kind: ClusterSecretStore
  target:
    name: my-secret
    creationPolicy: Owner
    deletionPolicy: Delete
  data:
  - secretKey: password
    remoteRef:
      key: my-1password-item
```

## Key Configuration Patterns

### HelmRelease Standard

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-release
  namespace: my-namespace
spec:
  interval: 5m
  chart:
    spec:
      chart: my-chart
      version: "1.0.0"  # Always pin explicit versions
      sourceRef:
        kind: HelmRepository
        name: my-repo
        namespace: my-namespace
  crds:
    adopt: true
    keep: false
  install:
    crds: CreateReplace
  upgrade:
    crds: CreateReplace
```

### Service Exposure via Traefik

Routes are defined via Kubernetes Ingress or Traefik IngressRoute resources. Example structure in `traefik/config/`:

- **middleware.yaml** - Request transformations (redirects, auth, etc.)
- **routes.yaml** - Service routing rules
- **services.yaml** - External service references
- **tls.yaml** - TLS certificate bindings

### Kustomization Composition

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: my-service
namespace: my-namespace

resources:
- init/kustomization.yaml
- release/kustomization.yaml

commonLabels:
  app.kubernetes.io/name: my-service
  app.kubernetes.io/instance: prod

dependsOn:
- name: cert-manager
  namespace: cert-manager
```

## Standards & Best Practices (from CRUSH.md)

### Kubernetes Resources

- **Mandatory fields:** apiVersion, kind, metadata.name, metadata.namespace
- **Labels:** `app.kubernetes.io/name`, `app.kubernetes.io/instance`
- **Resource specs:** CPU (e.g., 250m) and Memory (e.g., 512Mi) for both requests and limits

### Namespace Organization

- Each service has dedicated namespace
- Pattern: `<service-name>` (e.g., `traefik`, `cert-manager`, `nokiy-cms`)
- Isolation via NetworkPolicy (Cilium)

### Version Management

- Helm: Explicit version pinning (e.g., `version: "1.18.2"`)
- Images: Tag all images (no `latest` tags)
- Renovate Bot: Automatically updates pinned versions in manifests

### Git Workflow

- Renovate Bot creates automated PRs for dependency updates
- Single source of truth: Git repository
- Flux CD reconciles cluster state with Git every 5 minutes (HelmRelease default)

## Common Development Tasks

### Adding a New Service to Common

1. Create `common/<service>/` directory structure (init/, release/, config/, kustomization.yaml)
2. Define namespace and HelmRepository in `init/`
3. Define HelmRelease in `release/`
4. Add secrets to `external-secrets/` if needed
5. Create `kustomization.yaml` to compose layers
6. Add to environment kustomization files (`prod/kustomization.yaml`, `yuzu/kustomization.yaml`, etc.)

### Adding a Secret

1. Add item to 1Password
2. Create ExternalSecret in `common/<service>/secrets/`
3. Reference in HelmRelease `values` or volume mounts
4. Example: `common/external-secrets/secrets/1password-store.yaml`

### Environment-Specific Configuration

Use Kustomize patches in environment directories (`prod/`, `yuzu/`):

```yaml
# prod/<service>.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../common/<service>
patchesStrategicMerge:
- |-
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: my-release
  spec:
    values:
      replicas: 3
```

### Disabling Services

In environment kustomization files, comment out or remove service references:

```yaml
# prod/kustomization.yaml
resources:
- ../common/cert-manager
# - ../common/velero  # Disabled in prod
```

## Recent Development Focus

Latest commits show Traefik internal routing improvements:
- Internal TLS certificate management via cert-manager
- IngressRoute resource adoption
- Separation of internal vs. external entrypoints
- Tailscale integration optimization

## Tools & Technologies

- **Kubernetes:** K3s, Kubectl
- **GitOps:** Flux CD v2
- **Package Management:** Helm, Kustomize
- **Ingress Controller:** Traefik
- **TLS:** cert-manager with Let's Encrypt
- **Secrets:** External Secrets Operator + 1Password
- **Monitoring:** Signoz (in yuzu environment)
- **Backup:** Velero, Simple-backup
- **CI/CD:** Forgejo + Forgejo-runner
- **Database:** CloudNative PostgreSQL (CNPG)
- **VPN:** Tailscale
- **Network Policies:** Cilium
- do not add namespace field in Kustomization yaml
- all pvc init size should be 1Gi

---

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
