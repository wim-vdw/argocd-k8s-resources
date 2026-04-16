# Copilot Instructions for argocd-k8s-resources

## Repository Overview

This is the **source of truth for Kubernetes manifests and Helm charts** deployed via Argo CD. It is part of a three-repository Argo CD architecture:

1. **argocd-setup** - Argo CD installation and configuration
2. **argocd-apps** - Argo CD application definitions (references this repo)
3. **argocd-k8s-resources** - *(This repo)* Kubernetes manifests and Helm charts

**Active Usage**: Managed by Dependabot (GitHub Actions only), ~4000+ commits, actively maintained.

## Architecture & Key Conventions

### Repository Structure

```
manifests/
├── <application-name>/
│   └── overlays/
│       └── prod/
│           ├── kustomization.yaml      (Main Kustomize file)
│           ├── namespace.yaml
│           ├── sealedsecret.yaml      (Bitnami sealed-secrets)
│           ├── configmap.yaml
│           ├── deployment.yaml or *.yaml (custom resources)
│           └── values.yaml             (Helm chart overrides)
```

**Key Pattern**: All applications follow an **overlays-only structure** (no base/ directories). Each overlay is self-contained with:
- Resources (namespaces, storage classes, custom deployments)
- Sealed secrets for sensitive data
- Helm chart integration via kustomization helmCharts

### Manifest Organization

**23 Applications Deployed**:
- **Infrastructure**: cert-manager, sealed-secrets, actions-runner-controller, system-upgrade-controller
- **Monitoring**: alloy, fluentbit, monitoring stack
- **Applications**: redis, argo-workflows, keda, kyverno, traefik, metallb, tailscale, longhorn
- **Networking**: f5-nginx-ingress
- **Experimental**: test* directories for Crossplane (AWS/Azure)

### Kustomize + Helm Integration Pattern

Each `kustomization.yaml` typically:
1. Defines a namespace
2. Includes custom resources (ConfigMaps, Deployments, StorageClasses)
3. **Embeds a Helm chart** using `helmCharts` block:
   ```yaml
   helmCharts:
   - name: <chart-name>
     repo: https://charts.example.com
     version: X.Y.Z
     releaseName: <release-name>
     namespace: <namespace>
     valuesFile: values.yaml
   ```
4. Applies **kustomize patches** to override Helm chart outputs (e.g., Tailscale annotations)

**Why Patches?**: Used to inject custom annotations and metadata that the Helm charts don't expose as values. Example:
```yaml
- target:
    kind: Service
    name: redis-master
  patch: |-
    metadata:
      annotations:
        tailscale.com/expose: "true"
```

### Secrets Management

**Sealed Secrets (Bitnami)**: All sensitive data (passwords, API keys) stored as encrypted `SealedSecret` resources:
- Encrypted with cluster sealing key
- Only decryptable on the target cluster
- Part of each application's overlay
- Pattern file: `<app>/overlays/prod/sealedsecret.yaml`

### Environment Strategy

**Single Environment Only**: `prod` (no dev/staging overlays). All deployments target production cluster.

---

## Validation & Deployment

### Validation Commands

**Validate Kustomize builds** (dry-run before deploying):
```bash
# Validate a single application
kustomize build manifests/redis/overlays/prod > /tmp/redis.yaml
kubeval /tmp/redis.yaml

# Validate all manifests
for app in manifests/*/overlays/prod; do
  echo "Validating $(dirname $app)..."
  kustomize build "$app" | kubeval -
done
```

**Common validation tools** (not pre-installed, bring-your-own):
- `kustomize build` - Render manifests to stdout
- `kubeval` - Validate YAML against Kubernetes OpenAPI schema
- `kubesec scan` - Security audit of manifests (if used)

### Deployment (via Argo CD)

**Do NOT manually apply**. Argo CD watches this repository and auto-syncs changes:
1. Commit changes to main branch
2. Argo CD detects the update (webhook or polling)
3. Auto-syncs to cluster (if set to auto-sync)

See **argocd-apps** repo for application sync policies.

---

## Key Conventions & Patterns

### Naming & Organization

- **Application names** match directory names (e.g., `redis` app in `manifests/redis/`)
- **Helm releases** match application names (where applicable)
- **Namespaces** match or relate to application names (e.g., `namespace: redis`)
- **SealedSecrets** named with application context (e.g., `my-redis-secret`)

### Resource Files

**Standard overlay structure**:
- `namespace.yaml` - Always first, defines namespace with metadata
- `kustomization.yaml` - Central orchestrator, must come last logically in resources list
- `sealedsecret.yaml` - Encrypted credentials
- `configmap.yaml` - Non-sensitive config
- `<deployment|statefulset|...>.yaml` - Custom resources not provided by Helm
- `values.yaml` - Helm chart value overrides

### Helm Chart Patterns

**All Helm charts are from external repos** (Bitnami, official Helm charts, vendor repos). Common patterns:
- Helm chart version pinned explicitly (e.g., `version: 25.3.9`)
- `values.yaml` in the overlay **overrides** chart defaults (not comprehensive, only what differs)
- Kustomize patches add metadata not exposed by Helm values
- `includeCRDs: true` for charts that define Custom Resource Definitions

### Git Workflow

**Commit messages**: Brief, action-oriented, often version/image bumps:
- `update redis image to version X`
- `add simple StatefulSet configuration`
- `bump traefik chart version to X.Y.Z`

**Activity**: Regular dependency updates (via Dependabot and manual updates), ~1-5 commits/week

---

## Common Workflows

### Adding a New Application

1. Create directory: `manifests/<app-name>/overlays/prod/`
2. Create core files:
   - `kustomization.yaml` with helmCharts block
   - `namespace.yaml` with application namespace
   - `values.yaml` with Helm chart overrides
   - `sealedsecret.yaml` if secrets needed
3. Add custom resources (ConfigMaps, Deployments) as needed
4. Test: `kustomize build manifests/<app-name>/overlays/prod/ | kubeval -`
5. Commit and push; Argo CD will sync

### Updating Helm Chart Versions

1. Edit `manifests/<app>/overlays/prod/kustomization.yaml`
2. Update the `version:` field under `helmCharts:`
3. Review `values.yaml` for breaking changes in new release
4. Test build: `kustomize build manifests/<app>/overlays/prod/`
5. Commit with message like: `update <app> chart version to X.Y.Z`

### Modifying Application Configuration

1. **For Helm chart settings**: Edit `values.yaml` and Kustomize patches in `kustomization.yaml`
2. **For cluster-specific metadata**: Use kustomize patches (annotations, labels, resource limits)
3. **For secrets**: Edit `sealedsecret.yaml` (use `kubeseal` CLI to encrypt new values)
4. Test before committing

### Sealed Secrets Workflow

**To add/update a sealed secret**:
```bash
# 1. Create plaintext secret
echo -n "my-password" | kubectl create secret generic my-secret \
  --from-file=password=/dev/stdin \
  --dry-run=client -o yaml > /tmp/secret.yaml

# 2. Seal it (requires sealing key from cluster)
kubeseal -f /tmp/secret.yaml -w /tmp/sealedsecret.yaml

# 3. Copy encryptedData to the sealedsecret.yaml in the repo
```

---

## Testing & Quality

### Validation Before Commit

- **Kustomize builds cleanly**: `kustomize build <path> | kubeval -`
- **No hardcoded secrets** in YAML (use SealedSecrets)
- **Consistent naming**: Follow existing app naming patterns
- **Helm versions pinned**: No `latest` or floating versions

### CI/CD Pipelines

**Dependabot**: Watches GitHub Actions in `.github/workflows/` and suggests updates

**Manual GitHub Actions** (for future expansion):
- `self-hosted-work.yaml` - Self-hosted runner workflow
- `arc-demo.yaml` & `arc-demo-dockerindocker.yaml` - Actions Runner Controller demos
- `self-hosted-mac.yaml` - macOS self-hosted runner

These are operational workflows, not validation pipelines.

---

## Important Notes for Contributors

### DO

- Pin **all Helm chart versions** explicitly
- Use **SealedSecrets** for any sensitive data
- Follow the **overlay-only structure** (no base directories)
- Test manifests locally before pushing
- Keep commits focused and descriptive

### DON'T

- Commit plaintext secrets
- Use `latest` tags or floating chart versions
- Modify application names or namespaces lightly (impacts Argo CD sync)
- Manually apply to cluster (Argo CD is the source of truth)
- Add base/ directories (breaks the established pattern)

### Gotchas

- **Kustomize patch paths must match**: If a Helm chart changes the generated Service name, kustomize patches will fail silently
- **Sealed secret scoping**: A sealed secret is tied to the cluster it was created on; moving it requires re-sealing
- **Namespace conflicts**: Overlays define their own namespaces; don't assume cross-namespace references
- **Argo CD sync timing**: Changes may take several minutes to sync depending on poll interval

---

## Related Repositories

For full context on how manifests are deployed:
- **[argocd-setup](https://github.com/wim-vdw/argocd-setup)** - Argo CD installation, sealing key, cluster bootstrap
- **[argocd-apps](https://github.com/wim-vdw/argocd-apps)** - Application definitions pointing to this repo

This repository is the **working directory** for Kubernetes configurations; the other two are for cluster setup and application orchestration.
