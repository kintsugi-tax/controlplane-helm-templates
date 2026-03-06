# CLAUDE.md - Control Plane Helm Templates

## Overview

This repository contains Helm chart templates for the [Control Plane](https://controlplane.com) marketplace. Each chart deploys infrastructure services (databases, queues, caches, etc.) as Control Plane workloads. These are **not standard Kubernetes Helm charts** — they use Control Plane-specific resource kinds (`workload`, `gvc`, `identity`, `policy`, `secret`, `volumeset`, `domain`).

## Repository Structure

```
<chart-name>/
  icon.png              # Square icon, transparent background (required)
  versions/
    <semver>/           # e.g. 1.0.0, 2.1.0
      Chart.yaml        # Helm chart metadata
      values.yaml       # Default configuration values
      README.md         # Chart-specific documentation
      templates/
        _helpers.tpl    # Template helper definitions
        workload-*.yaml # Workload definitions
        identity.yaml   # Identity resources
        policy.yaml     # Policy bindings
        secret-*.yaml   # Secret definitions
        volumeset*.yaml # Volume set definitions
        gvc.yaml        # GVC definition (only for charts with createsGvc: true)
        domain-*.yaml   # Domain definitions (optional)
```

Some charts also have at root level:
- `RELEASES.md` — changelog (kafka, redis, redis-multi-location)
- `environments.yaml` — environment flags (test-app, test-app-2)

## Charts

31 charts covering databases (postgres, mysql, mariadb, mongodb, clickhouse, cockroach, tidb, postgis), caches (redis, redis-cluster, redis-multi-location), queues (kafka, rabbitmq, nats), search (opensearch, manticore), and apps (airflow, nginx, ollama, fusionauth, coraza, tyk, dbeaver, minio, etcd, ess, tailscale, cpln-task-runner, test-app, test-app-2).

## Key Conventions

### Versioning

- Each chart version lives in its own directory under `versions/` (e.g. `versions/3.1.0/`)
- Versions use semantic versioning
- New versions are full copies — not patches on prior versions
- `Chart.yaml` must have matching `version` field

### Chart.yaml Format

```yaml
apiVersion: v2    # or v3
name: <chart-name>
description: <description>
type: application
version: X.Y.Z
appVersion: "<app-version>"

annotations:
  created: "YYYY-MM-DD"
  lastModified: "YYYY-MM-DD"
  category: "database"       # or "app", "cache", etc.
  createsGvc: false          # true if chart creates its own GVC
  cpln/marketplace: "true"   # marketplace tagging
  cpln/marketplace-template: <chart-name>
  cpln/marketplace-template-version: X.Y.Z
```

### _helpers.tpl Structure

Helpers are organized into three sections:

1. **Resource Naming** — Named templates that generate deterministic resource names using `.Release.Name`:
   ```
   {{- define "<chart>.name" -}}
   {{- printf "%s-<suffix>" .Release.Name }}
   {{- end }}
   ```
   Convention: `<releaseName>-<service>-<resourceType>` (e.g. `myrel-pg-config`, `myrel-redis-vs`)

2. **Validation** — `fail`-based validation of configuration (e.g. backup provider checks, auth method exclusivity)

3. **Labeling** — Standard tag/label templates:
   - `<prefix>.chart` — chart name + version
   - `<prefix>.tags` — common labels including marketplace tags
   - `<prefix>.selectorLabels` — app name/instance labels

### Template Files

- Use Control Plane resource kinds: `kind: workload`, `kind: secret`, `kind: identity`, `kind: policy`, `kind: volumeset`, `kind: gvc`, `kind: domain`
- File naming: `<kind>-<qualifier>.yaml` (e.g. `workload-postgres.yaml`, `secret-config.yaml`)
- All resources set `gvc: {{ .Values.global.cpln.gvc }}` (injected by the platform)
- Tags are applied via the helpers include: `{{- include "<prefix>.tags" . | nindent 4 }}`
- Secrets are referenced in workloads via `cpln://secret/<name>.<key>`
- Volumes are referenced via `cpln://volumeset/<name>`

### values.yaml Conventions

- `image:` — container image with tag
- `resources:` — CPU/memory with `minCpu`, `minMemory`, `maxCpu`/`cpu`, `maxMemory`/`memory`
- `config:` — application-specific config (credentials, database names)
- `volumeset:` — storage capacity and autoscaling settings
- `internalAccess:` — firewall settings (`type: same-gvc | same-org | none | workload-list`)
- `backup:` — optional backup configuration with provider-specific settings (aws/gcp)
- Resource values are quoted in templates: `{{ .Values.resources.minCpu | quote }}`

### Workload Patterns

- Stateful workloads: `type: stateful` with volume mounts and autoscaling disabled (`metric: disabled`, `minScale: 1`, `maxScale: 1`)
- Identity linking: `identityLink: //identity/<identity-name>`
- Readiness probes with service-specific health checks
- `inheritEnv: false` for container isolation
- Firewall: `external.outboundAllowCIDR: [0.0.0.0/0]` with configurable internal access

### Policy Pattern

Policies grant `reveal` permission on secrets to identities:
```yaml
kind: policy
bindings:
  - permissions:
      - reveal
    principalLinks:
      - //gvc/<gvc>/identity/<identity>
targetKind: secret
targetLinks:
  - //secret/<secret-name>
```

## Development Workflow

### Adding a New Chart

1. Create `<chart-name>/icon.png` (square, transparent background)
2. Create `<chart-name>/versions/<version>/` with:
   - `Chart.yaml` with proper annotations
   - `values.yaml` with sensible defaults
   - `README.md` documenting the chart
   - `templates/_helpers.tpl` following the naming/validation/labeling pattern
   - Template files for each resource

### Adding a New Version

1. Copy the latest version directory to a new semver directory
2. Update `Chart.yaml` version and `lastModified`
3. Make changes to templates/values as needed

### Commit Style

Commit messages follow the pattern: `<short description> (#<PR-number>)`. Examples:
- `increased agent readiness probe threshold (#174)`
- `added autoscaling config in values file, quoted resource and admin secret (#169)`
- `adjusted naming to be more deterministic in helpers file (#167)`

Use lowercase, imperative or past tense. Keep descriptions concise.

## Build and Testing

There is no CI pipeline, test suite, or linting configuration in this repository. Charts are validated by the Control Plane platform when deployed. To validate templates locally, use `helm template`:

```sh
helm template <release-name> ./<chart>/versions/<version>/
```

## Important Notes

- `.gitignore` excludes `.DS_STORE` and `Chart.lock`
- Licensed under Apache License 2.0
- The `global.cpln.gvc` value is injected by the Control Plane platform at deploy time
- Do not remove or rename existing version directories — they may be referenced by deployed instances
