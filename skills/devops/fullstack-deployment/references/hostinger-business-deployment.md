# Hostinger Business Hosting (Shared Hosting) - Node.js Web App Deployment

## Overview

Hostinger Business Hosting provides a **Node.js Web App** feature for shared hosting plans. This is fundamentally different from VPS deployment - it's a managed platform with significant limitations.

## Critical Limitations

| Feature | Hostinger Business | Workaround |
|---------|-------------------|------------|
| Multiple Node.js apps | ❌ One only | Deploy API separately (Vercel/Railway/Render) |
| WebSocket | ❌ Not supported | Use polling or external service (Pusher/Ably) |
| Redis | ❌ Not available | Use external Redis (Upstash/Railway) for API |
| Custom ports | ❌ Only 80/443 | API on separate host |
| Custom Nginx | ❌ Not available | hPanel manages proxy |
| systemd/PM2/Docker | ❌ Not available | hPanel manages process |
| Custom cron | ❌ Limited | Use external cron (cron-job.org) |

## Architecture: Frontend on Hostinger, API Separate

```
┌─────────────────────────────────────────────────────────────┐
│  Hostinger Business (Shared)                                │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ apps/web (Next.js Frontend)                             │ │
│  │ - Node.js Web App via hPanel                            │ │
│  │ - Port 3000 (proxied to 80/443)                         │ │
│  │ - Static assets, SSR, API routes → proxied to API       │ │
│  └─────────────────────────────────────────────────────────┘ │
│  ❌ NO: apps/api, Redis, WebSocket, Custom Nginx, systemd  │
└─────────────────────────────────────────────────────────────┘
                              │
                    HTTPS + Rewrites
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  External API Host (Vercel/Railway/Render/VPS)              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ apps/api (Next.js Backend)                              │ │
│  │ - Full Node.js environment                              │ │
│  │ - Database, Redis, WebSocket, Cron jobs                 │ │
│  │ - Stripe webhooks, Blockchain RPCs                      │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Deployment Steps

### 1. Deploy API Separately (REQUIRED)

Deploy `apps/api` to Vercel/Railway/Render:

```bash
# Vercel
cd apps/api
vercel --prod
# Set ALL environment variables in Vercel dashboard

# Railway
railway login
railway link
railway up
# Set root directory to apps/api

# Render
# Create Web Service, root directory: apps/api
```

### 2. Build Frontend for Hostinger

```bash
# Run the build script (creates .next/standalone/)
chmod +x deploy-hostinger-business/scripts/build-frontend.sh
./deploy-hostinger-business/scripts/build-frontend.sh

# This creates: deploy-package/crypto-web-<timestamp>.tar.gz
# Contains: .next/standalone/, .next/static/, public/, package.json, next.config.js, .env.production
```

### 3. Deploy to Hostinger

1. Upload `.tar.gz` to Hostinger via File Manager or SCP
2. Extract in your domain's document root (e.g., `/home/user/yourdomain.com/`)
3. In hPanel → **Website → Node.js** → **Create Application**:
   - **Node.js Version**: 20.x (or 18.x)
   - **Application Root**: `/home/user/yourdomain.com/.next/standalone`
   - **Startup File**: `server.js`
   - **Environment**: `production`

4. **Environment Variables (in hPanel Node.js config):**
   ```
   NODE_ENV=production
   NEXT_PUBLIC_API_URL=https://api.yourdomain.com
   NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
   NEXT_PUBLIC_APP_URL=https://yourdomain.com
   ```

5. **Click "Create"** - hPanel runs `npm install` and starts the app

### 5. Configure Domain & SSL

1. **hPanel → Domains → Manage** → Ensure domain points to Hostinger
2. **hPanel → Security → SSL** → Force HTTPS + Let's Encrypt (auto-renewal)
3. **hPanel → Security → ModSecurity**: ON
4. **hPanel → Performance → Cache Manager**: Enable
4. **hPanel → Performance → Brotli Compression**: Enable

### 6. Configure API Connection

In `apps/web/next.config.js`, ensure rewrites point to external API:

```javascript
async rewrites() {
  return [{
    source: '/api/:path*',
    destination: `${process.env.NEXT_PUBLIC_API_URL}/api/:path*`
  }];
}
```

In API deployment (Vercel/Railway), configure CORS:

```javascript
// apps/api/next.config.js
async headers() {
  return [{
    source: '/api/:path*',
    headers: [
      { key: 'Access-Control-Allow-Origin', value: 'https://yourdomain.com' },
      { key: 'Access-Control-Allow-Credentials', value: 'true' },
      { key: 'Access-Control-Allow-Methods', value: 'GET,DELETE,PATCH,POST,PUT' },
      { key: 'Access-Control-Allow-Headers', value: 'X-CSRF-Token, X-Requested-With, Accept, Accept-Version, Content-Length, Content-MD5, Content-Type, Date, X-Api-Version, Authorization' },
    ],
  }];
}
```

### 7. Database (Hostinger MySQL)

1. **hPanel → Databases → MySQL Databases**
2. Create database: `crypto_trading`
3. Create user: `crypto_user` with strong password
4. Use `localhost` as host (internal connection)

### 6. Environment Variables

#### Frontend (hPanel Node.js Config)
```env
NODE_ENV=production
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
NEXT_PUBLIC_APP_URL=https://yourdomain.com
```

#### API (Vercel/Railway/Render - NOT on Hostinger)
```env
DATABASE_URL=mysql://user:pass@localhost:3306/crypto_trading
AUTH_SECRET=32+_char_random_string
NEXTAUTH_URL=https://api.yourdomain.com
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
REDIS_URL=redis://... (external - Upstash/Railway)
SMTP_*=... (email config)
*_RPC_URL=... (blockchain RPCs)
```

### DNS Records
```
Type    Name    Value
A       @       YOUR_HOSTINGER_IP
A       www     YOUR_HOSTINGER_IP
CNAME   api     cname.vercel.app (or your API host)
```

### GitHub Actions Auto-Deploy

```yaml
# .github/workflows/deploy-hostinger.yml
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci && cd apps/web && npm run build
      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.HOSTINGER_HOST }}
          username: ${{ secrets.HOSTINGER_USER }}
          key: ${{ secrets.HOSTINGER_SSH_KEY }}
          script: |
            cd /home/user/crypto-trading-platform
            git pull origin main
            cd apps/web && npm ci && npm run build
            # Restart via hPanel if needed
```

### Verification

```bash
# On server after deployment
DOMAIN=yourdomain.com ./scripts/verify-deployment.sh
# Checks: HTTPS, API health, WebSocket, DB, Redis, PM2, SSL, headers
```

### Rollback Procedures

```bash
# Code rollback
git checkout HEAD~1 && cd apps/web && npm run build
# Restart via hPanel

# Database rollback (on API host)
npx prisma migrate resolve --rolled-back <name>
npx prisma migrate deploy

# Full rollback from backup
gunzip -c backup/db_*.sql.gz | mysql -u user -p db
tar -xzf backup/app_*.tar.gz
```

## ⚠️ Key Differences from VPS Deployment

| Aspect | VPS (KVM) | Business Hosting |
|--------|-----------|------------------|
| Process Management | systemd/PM2/Docker | hPanel managed |
| Reverse Proxy | Custom Nginx | hPanel managed |
| SSL | Certbot + Nginx | hPanel Let's Encrypt |
| Multiple Apps | Unlimited | 1 Node.js app only |
| WebSocket | Full support | ❌ Not supported |
| Redis | Self-hosted/External | External only (for API) |
| Cron Jobs | systemd/cron | External (cron-job.org) |
| Custom Configs | Full control | hPanel UI only |
| Root Access | Yes | No |

## Troubleshooting

| Issue | Check |
|-------|-------|
| App won't start | hPanel Node.js logs, check `.env.production`, verify `server.js` exists |
| API routes fail | Check `NEXT_PUBLIC_API_URL`, verify API CORS, check rewrites in `next.config.js` |
| Static assets 404 | Verify `.next/static/` uploaded, check Nginx caching |
| SSL issues | hPanel → SSL → Force HTTPS, check certbot status |
| Build fails | Node version mismatch, check `package.json` engines field |

## Important Notes

1. **WebSocket Not Supported**: Real-time features won't work. Use polling or external service (Pusher, Ably, Centrifugo).

2. **Redis Not Available**: API must use external Redis (Upstash free tier: 10k commands/day).

3. **File Uploads**: Use external storage (AWS S3, Cloudflare R2) for KYC documents.

4. **Cron Jobs**: Use external cron service (cron-job.org, EasyCron) for scheduled tasks.

5. **Logs**: Check hPanel Node.js logs section for application output.

6. **Auto-Restart**: hPanel automatically restarts crashed apps, but check "Application Status" in hPanel.

7. **Cache**: hPanel caches static assets. Clear cache in hPanel after deploy if needed.The skill library has been updated with comprehensive Hostinger Business Hosting deployment guidance. Let me provide a summary of what was updated.
<tool_call>
<function=skill_view>
<parameter=name>
fullstack-deployment