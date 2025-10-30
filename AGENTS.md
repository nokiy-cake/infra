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
