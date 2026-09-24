# Vescrow — Production Deployment Guide

**Version**: 1.0  
**Last Updated**: June 2026  
**Audience**: DevOps Engineers, System Administrators, Backend Developers

---

## 1. Architecture & Strategy

Vescrow is compiled as a single stateless binary run within an Alpine Docker container. This simplifies scaling:
- **Stateless App Instances**: Can scale horizontally behind a load balancer (e.g., Nginx, Traefik, AWS ALB).
- **Persistent Storage**: Handled strictly by PostgreSQL and Redis.
- **File Uploads**: KYC documents and dispute evidence are stored in an S3-compatible object store (e.g., AWS S3, MinIO) rather than local disk.

```
                  ┌──────────────────┐
                  │   Client App     │
                  └────────┬─────────┘
                           │ HTTPS (TLS 1.3)
                           ▼
                  ┌──────────────────┐
                  │ Load Balancer    │
                  │ (Traefik/Nginx)  │
                  └────────┬─────────┘
                           │ HTTP
             ┌─────────────┴─────────────┐
             ▼                           ▼
     ┌───────────────┐           ┌───────────────┐
     │  Vescrow App  │           │  Vescrow App  │
     │  Instance 1   │           │  Instance 2   │
     └──────┬────┬───┘           └──────┬────┬───┘
            │    │                      │    │
   Postgres │    └────────┐    ┌────────┘    │ Redis
   (Write)  │             │    │             │ (Session/Rate limit)
            ▼             ▼    ▼             ▼
       ┌──────────┐     ┌─────────┐     ┌──────────┐
       │ Primary  │◄───►│ S3 Bucket│    │ Redis    │
       │ Postgres │     │ (KYC/   │     │ Cluster  │
       └────┬─────┘     │ Evidence│     └──────────┘
            │           └─────────┘
            ▼ Replication
       ┌──────────┐
       │ Replica  │
       │ Postgres │
       └──────────┘
```

---

## 2. Infrastructure Requirements

### 2.1 Hardware Specifications (Recommended Minimums)

| Tier | Target Traffic | CPU | RAM | Storage |
|------|----------------|-----|-----|---------|
| **Staging** | QA, testing | 1 vCPU | 1 GB | 10 GB SSD |
| **Production (Small)** | Up to 10k users | 2 vCPU | 4 GB | 50 GB NVMe |
| **Production (Medium)**| Up to 100k users | 4 vCPU | 8 GB | 100 GB NVMe |

### 2.2 Network Security

Only the following ports should be open to the public internet:
- `80/tcp` (redirect to 443)
- `443/tcp` (HTTPS traffic)
- `22/tcp` (SSH - locked down to trusted IPs or VPN only)

PostgreSQL (`5432`) and Redis (`6379`) must **never** be exposed publicly. They must only accept connections from the Vescrow app container network.

---

## 3. Deployment Configuration

### 3.1 Production Environment Variables

Generate a production `.env` file on the deployment host. Do **not** commit this file to source control.

```bash
# ── Server ──
PORT=3000
ENVIRONMENT=production
ALLOWED_ORIGINS=https://app.vescrow.com,https://admin.vescrow.com

# ── Database ──
DB_HOST=vescrow_db_primary
DB_PORT=5432
DB_USER=vescrow_prod_user
DB_PASSWORD=SecureProductionDatabasePassword99!
DB_NAME=vescrow_db
DB_SSL_MODE=require

# ── Redis ──
REDIS_URL=redis://:RedisProdAuthPassword@redis_prod_host:6379/0

# ── JWT (RS256) ──
JWT_PRIVATE_KEY_PATH=/etc/vescrow/keys/private.pem
JWT_PUBLIC_KEY_PATH=/etc/vescrow/keys/public.pem
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=720h

# ── Paystack API ──
PAYSTACK_SECRET_KEY=sk_live_e973bc884f22a...
PAYSTACK_PUBLIC_KEY=pk_live_f839ca889c22e...
PAYSTACK_WEBHOOK_SECRET=whsec_prod_e98c772c...

# ── SendGrid SMTP ──
SENDGRID_API_KEY=SG.live_key_goes_here...
SENDGRID_FROM_EMAIL=noreply@vescrow.com
SENDGRID_FROM_NAME=Vescrow Notifications

# ── Twilio SMS ──
TWILIO_ACCOUNT_SID=ACprod_sid_here
TWILIO_AUTH_TOKEN=prod_auth_token_here
TWILIO_PHONE_NUMBER=+234700VESCROW

# ── Encryption ──
# Must be exactly 32 bytes hex-encoded for AES-256-GCM
ENCRYPTION_KEY=64_character_hexadecimal_key_value_here...
```

### 3.2 Key Generation Utilities

#### Generating RSA Keys for JWT (RS256)
```bash
# Create directories
mkdir -p /etc/vescrow/keys

# Generate private key
openssl genrsa -out /etc/vescrow/keys/private.pem 4096
chmod 400 /etc/vescrow/keys/private.pem

# Extract public key
openssl rsa -in /etc/vescrow/keys/private.pem -pubout -out /etc/vescrow/keys/public.pem
chmod 444 /etc/vescrow/keys/public.pem
```

#### Generating AES-256 Symmetric Key
```bash
# Generate 32-byte (64 hex characters) key for PII encryption
openssl rand -hex 32
```

---

## 4. Production Docker Compose Spec

Create `/opt/vescrow/docker-compose.yml`:

```yaml
version: '3.8'

services:
  vescrow_app:
    image: ghcr.io/didihart/vescrow:latest
    container_name: vescrow_api
    restart: always
    expose:
      - "3000"
    environment:
      - PORT=3000
      - ENVIRONMENT=production
      - DB_HOST=vescrow_db
      - DB_USER=vescrow_prod_user
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=vescrow_db
      - REDIS_URL=redis://redis:6379/0
      - JWT_PRIVATE_KEY_PATH=/etc/vescrow/keys/private.pem
      - JWT_PUBLIC_KEY_PATH=/etc/vescrow/keys/public.pem
      - PAYSTACK_SECRET_KEY=${PAYSTACK_SECRET_KEY}
      - PAYSTACK_PUBLIC_KEY=${PAYSTACK_PUBLIC_KEY}
      - PAYSTACK_WEBHOOK_SECRET=${PAYSTACK_WEBHOOK_SECRET}
      - SENDGRID_API_KEY=${SENDGRID_API_KEY}
      - TWILIO_ACCOUNT_SID=${TWILIO_ACCOUNT_SID}
      - TWILIO_AUTH_TOKEN=${TWILIO_AUTH_TOKEN}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
    volumes:
      - /etc/vescrow/keys:/etc/vescrow/keys:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vescrow.rule=Host(`api.vescrow.com`)"
      - "traefik.http.routers.vescrow.entrypoints=websecure"
      - "traefik.http.routers.vescrow.tls.certresolver=letsencrypt"
      - "traefik.http.services.vescrow.loadbalancer.server.port=3000"
    depends_on:
      - vescrow_db
      - redis
    networks:
      - vescrow-net

  vescrow_db:
    image: postgres:15-alpine
    container_name: vescrow_db
    restart: always
    environment:
      - POSTGRES_DB=vescrow_db
      - POSTGRES_USER=vescrow_prod_user
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - vescrow-net
    command: >
      postgres 
      -c ssl=on 
      -c ssl_cert_file=/var/lib/postgresql/server.crt 
      -c ssl_key_file=/var/lib/postgresql/server.key
      -c max_connections=200
      -c shared_buffers=1GB

  redis:
    image: redis:7-alpine
    container_name: vescrow_redis
    restart: always
    command: redis-server --requirepass ${REDIS_PASSWORD} --appendonly yes
    volumes:
      - redisdata:/data
    networks:
      - vescrow-net

  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: always
    ports:
      - "80:80"
      - "443:443"
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@vescrow.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/traefik/letsencrypt:/letsencrypt
    networks:
      - vescrow-net

volumes:
  pgdata:
  redisdata:

networks:
  vescrow-net:
    driver: bridge
```

---

## 5. CI/CD Pipeline Blueprint (GitHub Actions)

Create `.github/workflows/deploy.yml`:

```yaml
name: Production Deployment

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/vescrow:latest
            ghcr.io/${{ github.repository }}/vescrow:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to VPS via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/vescrow
            docker compose pull vescrow_app
            docker compose up -d vescrow_app
            docker image prune -af
```

---

## 6. Backup & Recovery Policy

### 6.1 Database Backups (Daily Cron)

Create `/etc/cron.daily/vescrow-db-backup`:

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/vescrow"
DB_CONTAINER="vescrow_db"
DB_NAME="vescrow_db"
DB_USER="vescrow_prod_user"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
FILENAME="${BACKUP_DIR}/vescrow_${TIMESTAMP}.sql.gz"

mkdir -p $BACKUP_DIR

# Run pg_dump within container
docker exec -t $DB_CONTAINER pg_dump -U $DB_USER -d $DB_NAME | gzip > $FILENAME

# Restrict backup permissions
chmod 600 $FILENAME

# Delete backups older than 30 days
find $BACKUP_DIR -name "*.sql.gz" -type f -mtime +30 -delete

# Upload to S3-compliant offsite storage (using AWS CLI or rclone)
aws s3 cp $FILENAME s3://vescrow-backups/database/
```

### 6.2 Restoration Procedure

To restore from a backup:
```bash
# Decompress backup
gunzip vescrow_20260614_171500.sql.gz

# Drop and recreate database inside container
docker exec -it vescrow_db psql -U vescrow_prod_user -d postgres -c "DROP DATABASE vescrow_db;"
docker exec -it vescrow_db psql -U vescrow_prod_user -d postgres -c "CREATE DATABASE vescrow_db;"

# Restore schema and data
cat vescrow_20260614_171500.sql | docker exec -i vescrow_db psql -U vescrow_prod_user -d vescrow_db
```

---

## 7. Monitoring & Logging Configuration

### 7.1 Structured Logging

In production, log output should be directed to `stdout` in **JSON format** rather than plain text.
The logs are collected by Docker and forwarded to systemd journald or log shippers (e.g., Filebeat, Promtail).

Example log output:
```json
{"time":"2026-06-14T17:15:00Z","level":"INFO","module":"payment","message":"Paystack payment confirmed","escrow_id":"a11b4fa4-7e8c-4a3b-8d9e-1f2a3b4c5d6e","amount":350000000}
```

### 7.2 Health Check Endpoints

Vescrow must implement a dedicated health check endpoint:
- **Endpoint**: `/api/v1/health`
- **Purpose**: Checks connection to PostgreSQL and Redis.
- **Success Response (200 OK)**:
```json
{
  "status": "healthy",
  "database": "connected",
  "redis": "connected"
}
```
- **Failure Response (503 Service Unavailable)**:
```json
{
  "status": "unhealthy",
  "database": "disconnected",
  "redis": "connected"
}
```

---

## 8. Incident Response & Playbooks

### 8.1 Scenario: Double-Entry Ledger Discrepancy
1. **Detection**: Triggered by automated audit script executing daily, checking `SUM(credits) + SUM(debits) != 0`.
2. **Mitigation**:
   - Immediately run query to find the unbalanced escrow transactions.
   - Freeze withdrawals on affected seller wallets:
     ```sql
     UPDATE wallets SET is_frozen = true WHERE id = 'affected_wallet_id';
     ```
   - Notify the security team and audit officer.
3. **Resolution**: Manually verify bank transaction reference, correct the ledger through correcting entries (never UPDATE existing ledger rows), and unfreeze the wallet.

### 8.2 Scenario: Paystack Webhook Failure
1. **Detection**: User complains that their payment did not fund the escrow, or monitoring flags elevated 4xx/5xx on `/api/v1/payments/webhook/paystack`.
2. **Mitigation**:
   - Review `/opt/vescrow` application logs for verification or signature failure.
   - Trigger manual transaction verification using transaction reference:
     ```bash
     curl -X GET /api/v1/payments/verify/VESC_REF_9876543210 -H "Authorization: Bearer <ADMIN_JWT>"
     ```
3. **Resolution**: If Paystack service is down, retry the verification queue. If the webhook key was changed, update host environment variables and reload the container.
