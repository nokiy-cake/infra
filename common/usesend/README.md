# Usesend Deployment

This directory contains the Kubernetes manifests for deploying Usesend application.

## Architecture

- **PostgreSQL**: Managed by CloudNativePG (CNPG) operator
- **Redis**: Managed by Redis Operator
- **External S3**: Uses external S3-compatible storage service
- **Usesend App**: Main application container

## Prerequisites

Required operators must be installed:
- CloudNativePG (CNPG)
- Redis Operator
- External Secrets Operator
- Traefik (for ingress)
- cert-manager (for TLS)

## Secrets Configuration

Before deploying, create the following items in 1Password:

### 1. usesend-db
```
username: usesend
password: <random-password>
```

### 2. usesend-redis
```
password: <random-password>
```

### 3. usesend-s3
```
access-key: <s3-access-key>
secret-key: <s3-secret-key>
region: <s3-region>
endpoint: <s3-endpoint-url>
bucket: <s3-bucket-name>
```

### 4. usesend-app
```
nextauth-secret: <random-secret>
github-id: <your-github-oauth-id>
github-secret: <your-github-oauth-secret>
```

## Configuration

### Domain Names

Update the following files with your actual domain names:

1. **common/usesend/app/deployment.yaml**
   - Change `NEXTAUTH_URL` from `https://usesend.example.com` to your domain

2. **common/usesend/config/ingressroute.yaml**
   - Update Host() match:
     - `usesend.example.com` → Your main app domain

### S3 Configuration

Configure your external S3 service in 1Password (`usesend-s3` item):
- `access-key`: S3 access key ID
- `secret-key`: S3 secret access key
- `region`: S3 region (e.g., us-east-1, eu-west-1)
- `endpoint`: S3 endpoint URL (optional, for S3-compatible services like DigitalOcean Spaces, Cloudflare R2, etc.)
- `bucket`: S3 bucket name

For AWS S3, leave `endpoint` empty or set to AWS default.
For S3-compatible services, set the full endpoint URL (e.g., `https://nyc3.digitaloceanspaces.com`)

### SMTP Configuration (Optional)

If you want to use custom SMTP:
- Update `SMTP_HOST` and `SMTP_USER` in `app/deployment.yaml`
- Add SMTP password to secrets if needed

## Deployment

### Add to Environment

1. Edit `prod/kustomization.yaml` or `yuzu/kustomization.yaml`:
```yaml
resources:
- ../common/usesend
```

2. Commit and push to trigger Flux CD deployment

### Manual Validation

```bash
# Validate manifests
kubectl kustomize common/usesend --enable-helm

# Check deployment status
kubectl get pods -n usesend
kubectl get svc -n usesend
kubectl get ingressroute -n usesend

# View logs
kubectl logs -n usesend deployment/usesend
```

## Resource Requests/Limits

- **PostgreSQL**: 512Mi-1Gi memory, 250m-500m CPU
- **Redis**: 256Mi-512Mi memory, 100m-200m CPU
- **Usesend App**: 512Mi-1Gi memory, 250m-1000m CPU

## Storage

All PVCs use `local-path` StorageClass with 1Gi initial size:
- `usesend-db`: PostgreSQL data
- `usesend-redis`: Redis persistence

External S3 storage is used for application files and uploads.

## Troubleshooting

### Application fails to start

Check database and Redis connectivity:
```bash
kubectl logs -n usesend deployment/usesend -c wait-for-db
kubectl logs -n usesend deployment/usesend -c wait-for-redis
```

### S3 connection issues

1. Verify S3 credentials in 1Password (`usesend-s3` item)
2. Ensure bucket exists and is accessible
3. Check bucket permissions for read/write access
4. Verify endpoint URL is correct (for S3-compatible services)

### GitHub OAuth issues

Verify GitHub OAuth app configuration:
- Homepage URL: `https://usesend.example.com`
- Callback URL: `https://usesend.example.com/api/auth/callback/github`

## Access

- **Main App**: https://usesend.example.com
