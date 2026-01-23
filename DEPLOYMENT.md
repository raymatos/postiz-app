# Postiz Deployment Guide

## Overview
This deployment configuration sets up Postiz at `https://positiz.i9labs.com` with S3-compatible storage, Temporal workflow engine, and Spotlight debugging.

## Prerequisites
- Docker and Docker Compose installed
- Reverse proxy/tunnel configured on another server that forwards to this host
- S3-compatible storage endpoint accessible at `https://s3.shortrunprints.com`

## Configuration Files

### [`postiz.env`](postiz.env:1)
Contains all environment variables for the Postiz container. This file is:
- Mounted at `/config/postiz.env` inside the container (read-only)
- Loaded via `env_file` directive in docker-compose

**Key settings:**
- **URLs**: `https://positiz.i9labs.com` (managed by external reverse proxy)
- **Database**: PostgreSQL with randomly generated credentials
- **Storage**: S3-compatible via AWS SDK
- **JWT Secret**: Random 64-character string for session security

### [`docker-compose.yaml`](docker-compose.yaml:1)
Orchestrates all services required for Postiz.

## Services & Ports

| Service | Container Port | Host Port | Purpose |
|---------|---------------|-----------|---------|
| postiz | 5000 | 4007 | Main application (Postiz web UI + API) |
| postiz-postgres | 5432 | - | PostgreSQL database (internal only) |
| postiz-redis | 6379 | - | Redis cache (internal only) |
| temporal | 7233 | 7233 | Temporal workflow engine |
| temporal-ui | 8080 | 8080 | Temporal web UI for monitoring workflows |
| temporal-postgresql | 5432 | - | Temporal database (internal only) |
| temporal-elasticsearch | 9200 | - | Temporal search index (internal only) |
| spotlight | 8969 | 8969 | Sentry Spotlight debugging tool |

## Reverse Proxy Configuration

Your external reverse proxy should forward:
- `https://positiz.i9labs.com` → `http://<this-host>:4007`
- `https://positiz.i9labs.com/api` → `http://<this-host>:4007/api`

Ensure the proxy:
- Sets proper `X-Forwarded-For`, `X-Forwarded-Proto`, and `Host` headers
- Handles SSL/TLS termination
- Has appropriate timeouts for long-running API requests

## Deployment Steps

### 1. Initial Deployment
```bash
# Start all services
docker compose up -d

# Verify services are running
docker compose ps

# Check logs
docker compose logs -f postiz
```

### 2. Verify Database Initialization
```bash
# Check Postiz database
docker compose exec postiz-postgres psql -U postiz_prod_user -d postiz_production_db -c "\dt"

# Check Temporal database
docker compose exec temporal-postgresql psql -U temporal -c "\l"
```

### 3. Access the Application
- **Main App**: `https://positiz.i9labs.com` (via your reverse proxy)
- **Temporal UI**: `http://<host>:8080` (direct access for monitoring)
- **Spotlight**: `http://<host>:8969` (direct access for debugging)

### 4. Create First User
Navigate to `https://positiz.i9labs.com` and register the first account (unless `DISABLE_REGISTRATION=true` is set).

## Storage Verification

Test S3 connectivity:
```bash
# Watch Postiz logs while uploading a test image
docker compose logs -f postiz

# Upload an image via Postiz UI and verify it appears in your S3 bucket
```

## Managing Social Media Integrations

Social media credentials are **not configured** by default. To add platforms:

1. Edit [`postiz.env`](postiz.env:1)
2. Add API keys/secrets for desired platforms (lines 34-63)
3. Restart the Postiz container:
```bash
docker compose restart postiz
```

## Maintenance

### View Logs
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f postiz
docker compose logs -f temporal
```

### Update Postiz
```bash
# Pull latest image
docker compose pull postiz

# Recreate container with new image
docker compose up -d postiz
```

### Backup Database
```bash
# Backup Postiz database
docker compose exec postiz-postgres pg_dump -U postiz_prod_user postiz_production_db > postiz_backup_$(date +%Y%m%d).sql

# Backup Temporal database
docker compose exec temporal-postgresql pg_dump -U temporal > temporal_backup_$(date +%Y%m%d).sql
```

### Restore Database
```bash
# Restore Postiz database
cat postiz_backup_20260123.sql | docker compose exec -T postiz-postgres psql -U postiz_prod_user -d postiz_production_db

# Restore Temporal database
cat temporal_backup_20260123.sql | docker compose exec -T temporal-postgresql psql -U temporal
```

### Stop/Start Services
```bash
# Stop all services
docker compose down

# Start all services
docker compose up -d

# Stop while preserving volumes
docker compose stop

# Start stopped services
docker compose start
```

## Volumes

| Volume | Purpose |
|--------|---------|
| `postgres-volume` | Postiz PostgreSQL data |
| `postiz-redis-data` | Redis cache data |
| `postiz-config` | Postiz configuration directory |

**Note**: No uploads volume is needed since storage is handled by S3.

## Security Notes

1. **Database Credentials**: Randomly generated and stored in [`postiz.env`](postiz.env:1) and [`docker-compose.yaml`](docker-compose.yaml:27). Do not commit these to version control.
2. **JWT Secret**: 64-character random string in [`postiz.env`](postiz.env:1). Keep this secret.
3. **S3 Credentials**: Configured in [`postiz.env`](postiz.env:1), ensure the IAM user has minimal permissions (read/write to the specific bucket only).
4. **External Access**: Only port 4007 should be exposed via your reverse proxy. Temporal UI (8080) and Spotlight (8969) should be restricted to internal networks or VPN access only.

## Troubleshooting

### Postiz won't start
```bash
# Check dependencies
docker compose logs postiz-postgres
docker compose logs postiz-redis
docker compose logs temporal

# Verify DATABASE_URL in postiz.env matches postgres credentials
```

### Can't upload media
```bash
# Test S3 endpoint accessibility
curl -I https://s3.shortrunprints.com

# Verify S3 credentials in postiz.env
# Check Postiz logs for S3 errors
docker compose logs -f postiz | grep -i s3
```

### Temporal workflows not running
```bash
# Check Temporal service
docker compose logs temporal

# Check Temporal UI for errors
# Access: http://<host>:8080
```

## Network Architecture

```
External Network (Internet)
    ↓
Reverse Proxy/Tunnel (separate server)
    ↓ HTTPS → HTTP
Host Port 4007
    ↓
┌─────────────────────────────────────────┐
│ Docker Compose Services                 │
│                                          │
│ [postiz-network]                         │
│  - postiz (app)                          │
│  - postiz-postgres                       │
│  - postiz-redis                          │
│  - spotlight                             │
│                                          │
│ [temporal-network]                       │
│  - temporal                              │
│  - temporal-ui                           │
│  - temporal-postgresql                   │
│  - temporal-elasticsearch                │
│  - temporal-admin-tools                  │
│                                          │
│ [postiz joins both networks]             │
└─────────────────────────────────────────┘
```

## Environment Variables Reference

See [`postiz.env`](postiz.env:1) for the complete list. Key variables:

- `MAIN_URL` / `FRONTEND_URL`: Public URL users access
- `NEXT_PUBLIC_BACKEND_URL`: API endpoint URL
- `DATABASE_URL`: PostgreSQL connection string
- `REDIS_URL`: Redis connection string
- `TEMPORAL_ADDRESS`: Temporal server address
- `STORAGE_PROVIDER`: Set to `s3`
- `AWS_*`: S3 configuration (endpoint, bucket, credentials)
