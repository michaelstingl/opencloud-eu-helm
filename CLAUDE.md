# CLAUDE.md - AI Assistant Guide for OpenCloud Helm Charts

This document provides guidance for AI assistants working with the OpenCloud Helm Charts repository.

## Project Overview

This is a **community-maintained** Helm charts repository for deploying [OpenCloud](https://opencloud.eu) on Kubernetes. OpenCloud is a cloud collaboration platform (fork of ownCloud Infinite Scale) that provides file sync, share, and document collaboration features.

**Important**: These charts are NOT officially supported by OpenCloud GmbH. They are community-driven.

### Key Technologies
- **Helm 3.2.0+** for Kubernetes package management
- **Kubernetes 1.19+** target platform
- **Gateway API** for modern ingress routing (Cilium recommended)
- **Traditional Ingress** also supported (nginx, traefik, haproxy, contour, istio)

## Repository Structure

```
opencloud-eu-helm/
├── charts/
│   ├── opencloud/              # Production chart (recommended)
│   ├── opencloud-microservices/ # Pod-per-service architecture
│   └── opencloud-dev/          # Development/testing chart
├── deployments/
│   └── nats/                   # External NATS deployment configs
├── .github/
│   └── workflows/              # CI/CD for publishing to GHCR
├── CONTRIBUTING.md
├── MAINTAINERS.md
└── README.md
```

### Chart Comparison

| Chart | Use Case | Architecture | Complexity |
|-------|----------|--------------|------------|
| `opencloud` | Production | Single pod with all services | Low |
| `opencloud-microservices` | Fine-grained control | Pod-per-service | High |
| `opencloud-dev` | Development/testing | Single Docker container | Minimal |

## Key Components

The charts deploy these components:

1. **OpenCloud** - Main application (file sync, share, web interface)
2. **Keycloak** - Identity provider (OIDC authentication)
3. **PostgreSQL** - Database for Keycloak
4. **MinIO** - S3-compatible object storage (or external S3)
5. **Collabora/OnlyOffice** - Document editing
6. **Tika** - Full-text search extraction
7. **NATS** - Messaging system (microservices chart)

## Important Files

### Production Chart (`charts/opencloud/`)

| File | Purpose |
|------|---------|
| `Chart.yaml` | Chart metadata, version: 0.2.3, appVersion: latest |
| `values.yaml` | Default configuration values |
| `templates/_helpers/tpl.yaml` | Helm template helpers |
| `templates/opencloud/deployment.yaml` | Main OpenCloud deployment |
| `templates/gateway/*.yaml` | Gateway API HTTPRoute resources |
| `files/opencloud/*.yaml` | Config file templates (CSP, app-registry, search) |
| `files/keycloak/opencloud-realm.json.gotmpl` | Keycloak realm configuration |

### Microservices Chart (`charts/opencloud-microservices/`)

| File | Purpose |
|------|---------|
| `Chart.yaml` | Chart metadata, version: 0.3.10, appVersion: 4.0.0-rc.1 |
| `values.yaml` | Extensive service-by-service configuration |
| `templates/_common/*.tpl` | Shared template helpers |
| `deployments/helm/helmfile.yaml` | Helmfile for deployment |
| `deployments/timoni/` | Timoni + FluxCD deployment configs |

## Development Workflow

### Testing Changes

```bash
# Lint the chart
helm lint charts/opencloud

# Template rendering (dry-run)
helm template opencloud charts/opencloud --debug

# Install for testing
helm install opencloud charts/opencloud \
  --namespace opencloud \
  --create-namespace \
  --dry-run
```

### Version Management

- Charts follow [SemVer 2.0](https://semver.org/)
- Currently at `0.x.x` - breaking changes may occur
- Version in `Chart.yaml` is updated by CI on tag pushes
- Use Renovate comments for automated image tag updates:
  ```yaml
  # renovate: datasource=docker depName=opencloudeu/opencloud-rolling
  appVersion: latest
  ```

### CI/CD Pipeline

The GitHub Actions workflow (`.github/workflows/publish-helm-charts.yml`):
1. Triggers on pushes to `main` or version tags (`v*`)
2. Packages all three charts
3. Pushes to GitHub Container Registry (GHCR) as OCI artifacts
4. Registry: `ghcr.io/opencloud-eu/helm-charts/`

## Code Conventions

### Helm Template Patterns

1. **Naming**: Use helper templates for consistent naming
   ```yaml
   {{ include "opencloud.fullname" . }}        # e.g., release-opencloud
   {{ include "opencloud.opencloud.fullname" . }} # e.g., release-opencloud-opencloud
   ```

2. **Labels**: Standard Kubernetes labels via helpers
   ```yaml
   {{- include "opencloud.labels" . | nindent 4 }}
   {{- include "opencloud.selectorLabels" . | nindent 6 }}
   ```

3. **Image References**: Support global registry override
   ```yaml
   image: {{ include "opencloud.image" (dict "imageValues" .Values.image "global" .Values.global) | quote }}
   ```

4. **Conditional Resources**: Check `.Values.*.enabled`
   ```yaml
   {{- if .Values.opencloud.enabled }}
   ...
   {{- end }}
   ```

5. **Secrets**: Support both inline values and existing secrets
   ```yaml
   name: {{- if .Values.opencloud.existingSecret }}
           {{ .Values.opencloud.existingSecret }}
         {{- else }}
           {{ include "opencloud.opencloud.fullname" . }}
         {{- end }}
   ```

### Values.yaml Structure

```yaml
# Global settings apply across all components
global:
  domain:
    opencloud: cloud.opencloud.test
    keycloak: keycloak.opencloud.test
  tls:
    enabled: false
  oidc:
    issuer: ""
  storage:
    storageClass: ""
  image:
    registry: ""      # Global registry override
    pullPolicy: ""    # Global pull policy override

# Component-specific settings follow the pattern:
componentName:
  enabled: true
  image:
    registry: docker.io
    repository: org/image
    tag: "version"
    pullPolicy: IfNotPresent
  replicas: 1
  resources: {}
  persistence:
    enabled: true
    size: 10Gi
    storageClass: ""
    existingClaim: ""
  existingSecret: ""  # Use existing secret instead of inline values
```

## Security Considerations

### Default Credentials (MUST change in production)

| Component | Default User | Default Password | Values Path |
|-----------|--------------|------------------|-------------|
| Keycloak Admin | admin | admin | `keycloak.internal.adminPassword` |
| OpenCloud Admin | - | admin | `opencloud.adminPassword` |
| PostgreSQL | keycloak | keycloak | `postgres.password` |
| MinIO | opencloud | opencloud-secret-key | `opencloud.storage.s3.internal.rootPassword` |
| Collabora Admin | admin | admin | `collabora.admin.password` |

### Secret Management

- Always use `existingSecret` for production deployments
- Secrets should be created externally (e.g., via Sealed Secrets, External Secrets)
- Never commit real credentials to version control

## Common Tasks for AI Assistants

### Adding a New Configuration Option

1. Add to `values.yaml` with sensible defaults
2. Document in README.md with parameter table
3. Use in templates with proper conditionals
4. Consider backward compatibility

### Adding a New Component

1. Create directory under `templates/<component>/`
2. Add deployment.yaml, service.yaml, and other resources
3. Add configuration section in `values.yaml`
4. Add helper templates if needed
5. Update README.md documentation

### Modifying Ingress/Gateway Routes

- Gateway API routes: `templates/gateway/`
- Ingress resources: `templates/<component>/ingress.yaml`
- Check `httpRoute.enabled` and `ingress.enabled` flags
- Consider annotation presets for different controllers

### Debugging Template Issues

```bash
# Render specific template
helm template test charts/opencloud -s templates/opencloud/deployment.yaml

# Show computed values
helm show values charts/opencloud

# Debug with verbose output
helm template test charts/opencloud --debug 2>&1 | less
```

## Storage Modes

### S3 Storage (default)
- Internal MinIO: `opencloud.storage.s3.internal.enabled: true`
- External S3: `opencloud.storage.s3.external.enabled: true`

### PosixFS Storage
- Set `opencloud.storage.mode: posixfs`
- Requires NFS 4.2+ with flock and xattrs support
- Use `ReadWriteMany` for multiple replicas

## Scaling Limitations

These services CANNOT scale beyond 1 replica:
- **IDM** - Embedded LDAP doesn't support replication
- **IDP** - Depends on IDM
- **Search** - Uses BoltDB (single-writer)
- **OCM** - Federation state management
- **NATS** - Embedded; use external for HA

## Links and Resources

- [OpenCloud Documentation](https://docs.opencloud.eu)
- [Matrix Chat - Helm](https://matrix.to/#/%23opencloud-helm:matrix.org)
- [Matrix Chat - General](https://matrix.to/#/%23opencloud:matrix.org)
- [GitHub Discussions](https://github.com/orgs/opencloud-eu/discussions)
- [OCI Registry](https://github.com/orgs/opencloud-eu/packages?repo_name=helm)

## License

AGPLv3 - See LICENSE file for details.
