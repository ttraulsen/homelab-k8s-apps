# CLAUDE.md — homelab-k8s-apps

This repository manages **core applications** for the homelab. These are services required for daily
household use. Deploy after `homelab-k8s-infra` is running.

## Repository Role

Each application is a self-contained subdirectory with its own `helmfile.yaml`.
The root `helmfile.yaml` coordinates them all.

**Deployment order:** homelab-base → homelab-k8s-infra → **homelab-k8s-apps**

## Applications

| App | Namespace | Purpose | Chart |
|---|---|---|---|
| secrets-chart | `default` | Creates Kubernetes secrets from SOPS-encrypted values | custom |
| PostgreSQL | `database` | Shared database for all apps | bitnami/postgresql |
| Authentik | `auth` | Central SSO / identity provider (OIDC, OAuth2, SAML, passkeys) | authentik/authentik |
| Vaultwarden | `vault` | Password manager | custom/vaultwarden |
| Immich | `photos` | Photo library | immich (commented out — placeholder) |

**Deploy PostgreSQL and secrets-chart before other apps** — Authentik and Vaultwarden depend on them.

## Deployment

```bash
helmfile sync        # deploy / reconcile everything
helmfile diff        # preview changes
helmfile sync --selector app=authentik   # deploy a single app
```

Execution is **manual** — no automatic reconciliation (see ADR-0004 in homelab-base).

## Secrets Management

Uses **SOPS + age** encryption (see ADR-0007 in homelab-base).

- Plaintext: `secrets/*.yaml` — **never commit**, already in `.gitignore`
- Encrypted: `secrets/*.sops.yaml` — committed to Git
- Key file: `~/.config/age/key.txt` (set `SOPS_AGE_KEY_FILE`)

```bash
sops -e secrets/my-secret.yaml > secrets/my-secret.sops.yaml   # encrypt
sops -d secrets/my-secret.sops.yaml > secrets/my-secret.yaml   # decrypt
```

age public keys (both must be in `.sops.yaml` `creation_rules`):
- StarryNight: `age104xl76udz4syr3x4ltju2rcdd0mkvran3jv4520lyewgy2r0lgss9xly5z`
- hestia: `age14jnakt2vsmd37czh5wgn7ghkr276papa0vx2algs2lwhu9t3ufzsav9mlk`

## Database

A **single shared PostgreSQL instance** serves all apps (namespace: `database`).

- Service DNS: `postgres.database.svc.cluster.local`
- Credentials secret: `postgres-credentials`
- Each app gets its own database (e.g., `authentik`, `vaultwarden`)
- StorageClass: `local-path` (node-local NVMe)

When adding a new app that needs a database: add a new database entry to the PostgreSQL values and
update `postgres-credentials` in the secrets chart.

## Storage

Use `local-nas` StorageClass for application data (backed by NAS at `/mnt/nas/k8s`).
Use `local-path` only for ephemeral or cache data.

Typical PVC sizes used in this repo:
- Authentik: 50Gi on `local-nas`
- Vaultwarden: 20Gi on `local-nas`
- PostgreSQL: 10Gi on `local-path`

## Ingress & TLS

Traefik handles ingress (deployed in homelab-k8s-infra). cert-manager issues TLS certificates.

Conventions:
- Ingress host: `{service}.hestia` for local, dynamic DNS hostname for public-facing
- TLS secret name: `{service}-tls`
- Administrative/sensitive services: accessible via VPN only (see ADR-0008 in homelab-base)

## Resource Constraints

The node has 16GB RAM and a low-power CPU. Keep resource requests modest:
- Memory requests: 256Mi–512Mi for small services
- CPU requests: 100m–500m
- No high-availability replicas (single-node cluster by design)

## Adding a New Application

1. Create a subdirectory: `{appname}/`
2. Add `{appname}/helmfile.yaml` following the pattern of existing apps
3. Add encrypted secrets to `secrets/` if needed
4. Reference the new helmfile in the root `helmfile.yaml`
5. Add its Helm repository to the root `helmfile.yaml` if not already present
6. Document the app in this file's Applications table

## What NOT to Do

- Do not commit plaintext secret files
- Do not deploy apps before PostgreSQL and secrets-chart are ready
- Do not use high resource limits that could starve other services
- Do not add non-essential or experimental apps here — use homelab-k8s-apps-additional instead
- Do not change StorageClass reclaim policy without understanding data loss implications
