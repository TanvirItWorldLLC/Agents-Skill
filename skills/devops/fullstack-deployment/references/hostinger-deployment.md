# Hostinger VPS Deployment - Detailed Guide

## Prerequisites

- Hostinger VPS (KVM 2+ vCPU, 4GB+ RAM, 50GB+ NVMe SSD)
- Ubuntu 22.04 LTS or 24.04 LTS
- Domain name with A record pointing to VPS IP
- SSH access (root or sudo user)

## Complete Setup Script

```bash
#!/bin/bash
# Run as root or with sudo

# 1. System update
apt update && apt upgrade -y

# 2. Install Docker
curl -fsSL https://get.docker.com | sh
usermod -aG docker $USER

# 3. Install Docker Compose v2
COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep tag_name | cut -d '"' -f 4)
curl -L "https://github.com/docker/compose/releases/download/${COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# 4. Install Nginx + Certbot
apt install -y nginx certbot python3-certbot-nginx

# 5. Configure firewall
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable

# 6. Create deploy user (optional but recommended)
useradd -m -s /bin/bash -G docker deploy
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

## Nginx Configuration for WebSocket + Next.js

```nginx
# /etc/nginx/sites-available/yourdomain.com
# Rate limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    # SSL
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy strict-origin-when-cross-origin;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # Frontend (Next.js)
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 86400;
    }

    # API Backend
    location /api/ {
        limit_req zone=api burst=60 nodelay;
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 86400;
    }

    # Auth endpoints - stricter rate limiting
    location ~ ^/api/auth/(login|register) {
        limit_req zone=login burst=5 nodelay;
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket
    location /socket.io/ {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400;
    }

    # Static assets caching
    location /_next/static/ {
        proxy_pass http://localhost:3000;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    location /_next/image/ {
        proxy_pass http://localhost:3000;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}
```

## SSL Certificate Setup

```bash
# Get certificate
certbot --nginx -d yourdomain.com -d www.yourdomain.com --non-interactive --agree-tos -m admin@yourdomain.com

# Test auto-renewal
certbot renew --dry-run

# Enable auto-renewal timer
systemctl enable certbot.timer
systemctl start certbot.timer
```

## Application Deployment

```bash
# 1. Clone repo as deploy user
su - deploy
git clone https://github.com/TanvirItWorldLLC/Trading-website-.git
cd Trading-website-

# 2. Configure environment
cp apps/api/.env.example apps/api/.env
nano apps/api/.env  # Fill in all production values

# 3. Build and deploy
docker-compose -f docker-compose.prod.yml up -d --build

# 4. Run database migrations
docker-compose -f docker-compose.prod.yml exec api npx prisma migrate deploy

# 5. Verify deployment
docker-compose -f docker-compose.prod.yml ps
curl -f http://localhost:3001/api/health
curl -f http://localhost:3000
```

## Environment Variables Checklist

```env
# Database (use external managed DB for production)
DATABASE_URL="mysql://user:strong_password@db-host:3306/crypto_trading"

# Auth - CRITICAL: Generate strong random string
AUTH_SECRET="$(openssl rand -base64 32)"
NEXTAUTH_URL="https://yourdomain.com"

# Stripe (Live keys)
STRIPE_SECRET_KEY="sk_live_..."
STRIPE_WEBHOOK_SECRET="whsec_..."
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY="pk_live_..."

# Google OAuth
GOOGLE_CLIENT_ID="..."
GOOGLE_CLIENT_SECRET="..."

# Redis
REDIS_URL="redis://redis:6379"

# Email
SMTP_HOST="smtp.sendgrid.net"
SMTP_PORT="587"
SMTP_USER="apikey"
SMTP_PASSWORD="SG...."
EMAIL_FROM="noreply@yourdomain.com"

# Blockchain RPCs (use paid providers for production)
ETH_RPC_URL="https://eth-mainnet.g.alchemy.com/v2/..."
BSC_RPC_URL="https://bsc-dataseed.binance.org/"
TRON_RPC_URL="https://api.trongrid.io"
```

## Monitoring & Maintenance

```bash
# View logs
docker-compose -f docker-compose.prod.yml logs -f --tail=100

# Health checks
curl -f https://yourdomain.com/api/health
curl -f https://yourdomain.com

# Database backup (run daily via cron)
docker-compose -f docker-compose.prod.yml exec -T db mysqldump -u root -p"$DB_ROOT_PASSWORD" crypto_trading > backup_$(date +%F).sql

# Update deployment
cd /home/deploy/Trading-website-
git pull
docker-compose -f docker-compose.prod.yml up -d --build
docker-compose -f docker-compose.prod.yml exec api npx prisma migrate deploy
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Container restarting | `docker-compose logs api` - check for missing env vars |
| 502 Bad Gateway | Check if containers are healthy: `docker-compose ps` |
| SSL cert errors | `certbot renew --force-renewal` |
| DB connection failed | Verify DATABASE_URL, check `docker-compose logs db` |
| Prisma migrate fails | Ensure DB is ready, run manually: `docker-compose exec api npx prisma migrate deploy` |
| High memory usage | Add swap: `fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile` |