# Linkding Deployment

Self-hosted bookmark manager with OIDC SSO support and automated SQLite backups via Litestream.

## Directory Structure

```
linkding/
├── init/              # Namespace initialization
├── release/           # Deployment, Service, PVC resources
├── config/            # Ingress configuration
├── secrets/           # OIDC and Litestream backup secrets
├── OIDC_SETUP.md      # OpenID Connect configuration guide
├── LITESTREAM_SETUP.md # SQLite backup configuration guide
└── README.md          # This file
```

## Features

### Core Features
- Clean UI optimized for bookmark management
- Tag-based organization
- Bulk editing, Markdown notes, read-it-later functionality
- Share bookmarks with other users or guests
- Progressive Web App (PWA) support
- REST API for 3rd party apps
- Admin panel

### Deployment Features
- **Automatic SQLite Backup**: Continuous replication to S3-compatible storage via Litestream
- **OIDC/SSO Support**: Single sign-on integration with Authelia, Authentik, etc.
- **High Availability**: Init containers for automatic database restore on pod failure
- **TLS**: Automatic certificate management with cert-manager

## Deployment Details

### Service Endpoints
- **URL**: https://link.nokiy.net
- **Port**: 9090 (internal), 80 (service), 443 (ingress)

### Storage
- **Database**: SQLite at `/etc/linkding/data/db.sqlite3`
- **PVC**: `linkding-data` (local-path storage class, 1Gi)
- **Backup**: S3-compatible storage at `backup.nokiy.net`

### Container Architecture

#### Init Containers
1. **litestream-restore** (0.5.2)
   - Restores database from S3 if local doesn't exist
   - `-if-db-not-exists` ensures first startup creates fresh DB

2. **fix-permissions** (busybox)
   - Fixes file ownership after restore
   - Changes ownership from root to 1000:1000

#### Application Containers
1. **litestream** (0.5.2)
   - Sidecar for continuous database backup
   - Monitors `/etc/linkding/data/db.sqlite3`
   - Replicates to `backup.nokiy.net/backup/linkding-litestream`

2. **linkding** (1.44.1)
   - Main application container
   - Shares data volume with litestream

## Configuration

### OIDC Setup

To enable OpenID Connect authentication:

1. Configure OIDC provider with redirect URI: `https://link.nokiy.net/oidc/callback/`
2. Update `linkding-oidc` secret with provider endpoints
3. Set `LD_ENABLE_OIDC=true` environment variable

See `OIDC_SETUP.md` for detailed setup instructions.

### Litestream Backup Configuration

Litestream is automatically configured to:
- Pull credentials from secret store (`backup.nokiy.net`)
- Backup to S3 bucket: `backup`
- Path in bucket: `linkding-litestream`
- Auto-restore on pod startup

See `LITESTREAM_SETUP.md` for backup management and recovery procedures.

## Deployment Dependencies

Flux deployment order:
1. **linkding-init** → Creates namespace (depends on k3s)
2. **linkding-secrets** → Pulls ExternalSecrets (depends on external-secrets)
3. **linkding-release** → Deploys app (depends on linkding-secrets)
4. **linkding-config** → Sets up Ingress (depends on traefik, cert-manager)

## Volume Mounts

| Container | Path | Mount | Purpose |
|-----------|------|-------|---------|
| litestream-restore | /etc/linkding/data | data PVC | Database restore |
| litestream-restore | /etc/litestream.yml | secret | Config file |
| litestream | /etc/linkding/data | data PVC | Monitor & backup |
| litestream | /etc/litestream.yml | secret | Config file |
| linkding | /etc/linkding/data | data PVC | Application data |

## Environment Variables

### Core Settings
- `LD_SERVER_PORT`: 9090
- `LD_ENABLE_REGISTRATION`: false
- `LD_ENABLE_ADMIN_PANEL`: true
- `LD_CSRF_TRUSTED_ORIGINS`: https://link.nokiy.net

### OIDC Configuration (Optional)
- `LD_ENABLE_OIDC`: false (set to true to enable)
- `OIDC_RP_CLIENT_ID`: From secret
- `OIDC_RP_CLIENT_SECRET`: From secret
- `OIDC_OP_AUTHORIZATION_ENDPOINT`: From secret
- `OIDC_OP_TOKEN_ENDPOINT`: From secret
- `OIDC_OP_USER_ENDPOINT`: From secret
- `OIDC_OP_JWKS_ENDPOINT`: From secret

## Troubleshooting

### Database Not Initializing
Check init container logs:
```bash
kubectl -n linkding logs deployment/linkding -c litestream-restore
```

### Backup Not Working
Check litestream sidecar logs:
```bash
kubectl -n linkding logs deployment/linkding -c litestream
```

### Permission Issues
Check fix-permissions init container:
```bash
kubectl -n linkding logs deployment/linkding -c fix-permissions
```

### OIDC Login Failing
1. Verify `LD_ENABLE_OIDC=true`
2. Check OIDC provider configuration
3. Ensure CSRF trusted origins include auth server

## Resource Requests/Limits

| Container | CPU (req/lim) | Memory (req/lim) |
|-----------|---------------|------------------|
| litestream-restore | 10m/1000m | 32Mi/512Mi |
| fix-permissions | 10m/100m | 16Mi/32Mi |
| litestream | 10m/1000m | 32Mi/512Mi |
| linkding | 100m/500m | 256Mi/512Mi |

## References

- [Linkding GitHub](https://github.com/sissbruecker/linkding)
- [Linkding Documentation](https://linkding.link/)
- [Litestream Documentation](https://litestream.io/)
- [OpenID Connect Spec](https://openid.net/specs/openid-connect-core-1_0.html)
