---
name: fullstack-deployment
description: Deploy Next.js/Docker/Prisma monorepos to Hostinger/Vercel.
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [deployment, docker, docker-compose, nextjs, prisma, turborepo, hostinger, vercel, monorepo]
    related_skills: [plan, systematic-debugging, taste-learning]
---

# Full-Stack Production Deployment

**Use when:** Deploying production-ready full-stack applications with Docker, including Next.js frontends/backends, databases, and monorepo architectures to Hostinger VPS, Vercel, or AWS.

## Architecture Decision: Single App vs Monorepo

Before scaffolding, ask the user. Default assumptions are wrong:

| App shape | Pick when |
|---|---|
| **Single Next.js app** (App Router + API route handlers) | One product, one team, no separate runtime requirements, no Socket.io / worker tier. Default for portfolios, CMS, dashboards, MVPs. |
| **Turborepo monorepo** (`apps/web` + `apps/api`) | Two products on different runtimes, real-time features, separate deploy cadences, shared packages consumed by 2+ apps. |
| **Next.js frontend + separate Express API** | Existing Express backend must stay; legacy code in the API tier; team split. |

**Why this matters:** the Turborepo scaffolding adds a shared package, two Dockerfiles, tarball packaging for Vercel, and ~30% more surface area. For a single Next.js full-stack app (App Router + `app/api/*` route handlers + Prisma + MySQL), you skip all of that and run one `next start` behind Nginx. The user chose single-app for a portfolio rebuild after seeing options — confirm before defaulting.

## Core Principles

- **Multi-stage Docker builds** — Separate deps, builder, and runner stages for minimal images
- **Standalone Next.js output** — Use `output: 'standalone'` for self-contained deployments
- **Health checks everywhere** — Database, Redis, API, and Web services all need readiness probes
- **Environment isolation** — `.env.example` templates, never commit secrets
- **ARM64 compatibility** — Next.js SWC lacks ARM64 binaries; use Babel transpilation as fallback
- **Verify what you can; document what you can't** — see Verification section below

## Monorepo Deployment (Turborepo)

### Structure
```
project/
├── apps/
│   ├── api/          # Next.js API backend
│   └── web/          # Next.js Frontend
├── packages/
│   └── shared/       # Shared types, schemas, utils
├── docker-compose.prod.yml
├── turbo.json
└── package.json
```

### Turborepo 2.x Config (`turbo.json`)
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "db:generate": { "outputs": ["prisma/client/**"] }
  }
}
```

### Shared Package Distribution
For Vercel (no workspace support), package shared as tarball:
```bash
cd packages/shared && npm pack
# Copy .tgz to apps/web/ and apps/api/
# Reference in package.json: "file:./crypto-trading-platform-shared-0.0.1.tgz"
```

## Next.js Production Dockerfile

### Multi-Stage Pattern
```dockerfile
# Base
FROM node:20-alpine AS base

# Dependencies
FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json* ./
COPY packages/shared ./packages/shared
COPY apps/api/package.json ./apps/api/
COPY apps/api/shared.tgz ./apps/api/
RUN npm ci

# Generate Prisma Client
COPY apps/api/prisma ./apps/api/prisma/
RUN cd apps/api && npx prisma generate

# Builder
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build --filter=api

# Runner
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder --chown=nextjs:nodejs /app/apps/api/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/api/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/apps/api/prisma ./prisma
USER nextjs
EXPOSE 3001
CMD ["node", "server.js"]
```

### Critical Next.js Config
```javascript
// next.config.js
const nextConfig = {
  output: 'standalone',  // Essential for Docker
  swcMinify: false,      // Disable for ARM64
  experimental: {
    forceSwcTransforms: false,
    useWasmBinary: true,  // WASM fallback for ARM64
  },
  webpack: (config) => {
    // ARM64: Disable SWC, use Babel
    config.resolve.alias['@next/swc'] = false;
    config.resolve.alias['@next/swc-linux-arm64'] = false;
    config.resolve.alias['@next/swc-android-arm64'] = false;
    config.module.rules.push({
      test: /\.(js|jsx|ts|tsx)$/,
      use: [{ loader: 'babel-loader', options: { presets: ['next/babel', '@babel/preset-env', '@babel/preset-typescript'] } }],
      exclude: /node_modules/,
    });
    return config;
  },
};
module.exports = nextConfig;
```

## Docker Compose Production

```yaml
version: '3.8'
services:
  db:
    image: mariadb:10.11
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: crypto_trading
    volumes: [db_data:/var/lib/mysql]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s

  redis:
    image: redis:7-alpine
    volumes: [redis_data:/data]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

  api:
    build: ./apps/api
    environment:
      DATABASE_URL: mysql://${DB_USER}:${DB_PASSWORD}@db:3306/crypto_trading
      REDIS_URL: redis://redis:6379
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_healthy }

  web:
    build: ./apps/web
    environment:
      NEXT_PUBLIC_API_URL: https://${DOMAIN}/api
    depends_on: [api]

volumes: [db_data, redis_data]
```

## Environment Management

### `.env.example` Template
```env
# Database
DATABASE_URL="mysql://user:pass@host:3306/db"

# Auth
AUTH_SECRET="32-char-minimum-random-string"
NEXTAUTH_URL="https://yourdomain.com"

# External Services
STRIPE_SECRET_KEY="sk_live_..."
GOOGLE_CLIENT_ID="..."
GOOGLE_CLIENT_SECRET="..."
REDIS_URL="redis://redis:6379"

# Blockchain RPCs
ETH_RPC_URL="https://eth-mainnet.g.alchemy.com/v2/..."
```

### Never Commit
- `.env` files
- `.env.local`, `.env.production`
- Any file with real credentials

## Hostinger VPS Deployment (KVM - Full Control)

### Server Setup
```bash
# Install Docker
curl -fsSL https://get.docker.com | sudo sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# SSL with Let's Encrypt
sudo apt install -y nginx certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

### Nginx Config
```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://localhost:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /socket.io/ {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Deploy Commands
```bash
git clone <repo>
cd <repo>
cp apps/api/.env.example apps/api/.env
# Edit .env with production values
docker-compose -f docker-compose.prod.yml up -d --build
docker-compose -f docker-compose.prod.yml exec api npx prisma migrate deploy
```

## Hostinger Business Hosting (Shared Hosting - Node.js Web App)

### Critical Limitations
| Feature | Hostinger Business | Workaround |
|---------|-------------------|------------|
| Multiple Node.js apps | ❌ One only | Deploy API separately (Vercel/Railway/Render) |
| WebSocket | ❌ Not supported | Use polling or external service (Pusher/Ably) |
| Redis | ❌ Not available | Use external Redis (Upstash/Railway) for API |
| Custom ports | ❌ Only 80/443 | API on separate host |
| Custom Nginx | ❌ Not available | hPanel manages proxy |
| systemd/PM2/Docker | ❌ Not available | hPanel manages process |
| Custom cron | ❌ Limited | Use external cron (cron-job.org) |

### Architecture: Frontend on Hostinger, API Separate
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

### Frontend Deployment Steps

#### 1. Prepare Frontend Build
```bash
# Local or CI/CD
cd apps/web
npm ci
npm run build  # Creates .next/standalone/ with server.js

# Package for upload
tar -czf deploy-web.tar.gz \
  .next/standalone/ \
  .next/static/ \
  public/ \
  package.json \
  next.config.js \
  .env.production
```

#### 2. hPanel Node.js App Configuration
1. **hPanel → Website → Node.js → Create Application**
2. **Settings:**
   - Node.js Version: **20.x** (or 18.x)
   - Application Root: `/home/user/yourdomain.com/.next/standalone`
   - Startup File: `server.js`
   - Environment: `production`

3. **Environment Variables (in hPanel Node.js config):**
   ```env
   NODE_ENV=production
   NEXT_PUBLIC_API_URL=https://api.yourdomain.com
   NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
   NEXT_PUBLIC_APP_URL=https://yourdomain.com
   ```

#### 4. API Deployment (Separate - Required)
Deploy `apps/api` to **Vercel** (recommended), **Railway**, **Render**, or **VPS**:
```bash
# Vercel example
cd apps/api
vercel --prod
# Set ALL environment variables in Vercel dashboard
```

### hPanel Configuration

#### Domain & SSL
- **hPanel → Domains → Manage** → Ensure domain points to Hostinger
- **hPanel → Security → SSL** → Force HTTPS + Let's Encrypt (auto-renewal)

#### Security Settings
- **Security → ModSecurity**: ON
- **Security → Hotlink Protection**: ON
- **Advanced → HTTP/2**: Enable

#### Performance
- **Performance → Cache Manager**: Enable
- **Performance → Brotli Compression**: Enable

#### API Connection
In `apps/web/next.config.js`, ensure rewrites point to external API:
```javascript
async rewrites() {
  return [{
    source: '/api/:path*',
    destination: `${process.env.NEXT_PUBLIC_API_URL}/api/:path*`
  }];
}
```

### Database (Hostinger MySQL)
1. **hPanel → Databases → MySQL Databases**
2. Create database: `crypto_trading`
2. Create user: `crypto_user` with strong password
3. Host: `localhost` (internal connection)

### Environment Variables

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
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
REDIS_URL=redis://... (Upstash/Railway)
SMTP_*=...
*_RPC_URL=...
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

### Verification Script
```bash
# On server after deployment
DOMAIN=yourdomain.com ./verify-deployment.sh
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

---

## Vercel Monorepo Deployment

## Vercel Monorepo Deployment

### Web App (Frontend)
- Root Directory: `apps/web`
- Build Command: `npm run build`
- Environment Variables: Add in Vercel Dashboard

### API App (Backend)
- Root Directory: `apps/api`
- Build Command: `npm run build`
- Same environment variables

### Shared Package
Package as tarball and reference locally (Vercel doesn't support workspaces):
```json
"dependencies": {
  "@myapp/shared": "file:./shared.tgz"
}
```

## Prisma Production

### Generate Client
```bash
npx prisma generate
```

### Migrations
```bash
# Development
npx prisma migrate dev

# Production
npx prisma migrate deploy
```

### Database URL Format
```
mysql://user:password@host:3306/database?sslmode=disable
postgresql://user:password@host:5432/database?schema=public
```

## Prisma Version Migration (5 → 6 → 7 → 8)

Prisma's command surface, config format, and engine model have changed substantially across recent majors. **Pin the version you wrote against** — the migration cost is non-trivial.

### When the project was scaffolded

| Prisma version | Engine model | Config file | Deploy command | Notes |
|----------------|-------------|-------------|----------------|-------|
| 5.x | Native `.so.node` query engine (x86_64 or ARM64 per `binaryTargets`) | `prisma/schema.prisma` only — `url` in `datasource` block | N/A in core; use `bunx @prisma/cli app deploy` for Prisma Compute | Engine binary must be loadable on the runtime host; on ARM64 (Termux, Graviton) requires the fallback pattern in `references/prisma-arm64-dev.md` |
| 6.x | Same as 5.x + introduces `queryCompiler` preview (WASM-based, runs anywhere) | Same as 5.x | Same | `queryCompiler` removes the native engine dependency when used |
| **7.x (stable, recommended)** | Query Compiler (WASM) by default — no native engine binary needed at runtime | `prisma/schema.prisma` + **`prisma.config.ts`** at project root; `url` lives in the config file, not the schema | `bunx @prisma/cli@latest app deploy --project ... --branch ... --env .env` | `prisma db push` works. `@prisma/client@7.10.0` is the current stable. The `lib/prisma.local.ts` fallback is still useful for environments where even Prisma 7's CLI binary is missing (e.g. Termux). |
| 8.x (Composer, `8.0.0-rc.12` and up) | Still Query Compiler, but the architecture is "Prisma Composer" — separate `contract` workflow, `db` subcommands replaced `generate`/`push` | `prisma.config.ts` with `definePrismaConfig` from `@prisma/cli-engine`; nested `orm: { schema, migrations, datasource }` block | `prisma deploy <entry>` — expects a runtime adapter module that exports a fetch handler. Next.js 14 doesn't ship one; you must author `app/prisma.compute.ts` (or similar). | `@prisma/client@8.x` not yet published as of 8.0.0-rc.12. Stick to `@prisma/client@7.10.0` even when using the v8 CLI — they're version-independent. |

### Choosing what to pin
- **Stay on Prisma 5.x**: only if you have a custom `lib/prisma.local.ts` fallback in place AND the project is x86_64-only. Prisma 5's behavior is well-understood; the version is the last one with a fully native engine model.
- **Migrate to Prisma 7.10.0 (recommended for new projects)**: gives you the WASM Query Compiler (no native engine), `prisma.config.ts`, and the v7-style `app deploy` command. Works on any host without the local-fallback dance.
- **Migrate to Prisma 8 (only if you need Composer features)**: bigger refactor — `prisma.config.ts` shape changes (nested `orm:` block, `definePrismaConfig` from a different package), `prisma generate` is gone (use `prisma db init`/`sign`/`update`), and you need a runtime entry module for `prisma deploy`. Defer until Composer stabilizes out of `rc.*`.

### Migrating from 5/6 to 7 (the realistic case)

1. Update `prisma/schema.prisma`: remove the `url` line from `datasource`. Keep `provider` and `relationMode`.
2. Create `prisma.config.ts` at the project root:
   ```ts
   import 'dotenv/config';
   import path from 'node:path';
   import { fileURLToPath } from 'node:url';
   import { defineConfig } from 'prisma/config';
   const __dirname = path.dirname(fileURLToPath(import.meta.url));
   export default defineConfig({
     schema: path.join(__dirname, 'prisma', 'schema.prisma'),
     migrations: { path: path.join(__dirname, 'prisma', 'migrations') },
     experimental: { adapter: true },
     datasource: { url: process.env.DATABASE_URL! },
   });
   ```
3. Update `package.json`:
   - `prisma` and `@prisma/client` → `^7.10.0` (or pinned `7.10.0`)
   - Scripts: `prisma generate` still works, but the `postinstall: "prisma generate"` hook now needs to be guarded: `"postinstall": "prisma generate || true"` because generate will fail on hosts without the engine binary. With Prisma 7's Query Compiler it should actually succeed, but the `|| true` is defensive.
4. Run `npm install --ignore-scripts` (to avoid the postinstall hook) then `npx prisma generate` to populate `node_modules/.prisma/client/`.
5. Delete the `lib/prisma.local.ts` fallback if you no longer need it — Prisma 7 works natively on ARM64 via WASM.

### Migrating from 7 to 8 (skip unless you need Composer)

1. Update `prisma.config.ts`:
   ```ts
   import { definePrismaConfig } from '@prisma/cli-engine';  // NOT prisma/config
   export default definePrismaConfig({
     orm: {  // nested — top-level keys are rejected with CLI.CONFIG_UNKNOWN_SECTION
       schema: 'prisma/schema.prisma',
       migrations: { path: 'prisma/migrations' },
       datasource: { url: process.env.DATABASE_URL! },
     },
   });
   ```
2. Install Prisma 8: `npm install prisma@8.x @prisma/cli@8.x` (keep `@prisma/client@7.10.0`).
3. Replace `prisma generate` with `prisma db sign` (or `prisma db init` to bootstrap). The output no longer lives in `node_modules/.prisma/client/default.js`; the new client API is the same but generation produces different artifacts.
4. Replace `bunx @prisma/cli app deploy --project ... --branch ...` with `prisma deploy <entry>` and author `<entry>` as a fetch handler that wraps your app.
5. CI deploy: `prisma deploy` is now the single command. `prisma db migrate deploy` is still available for migrations but `prisma db sign`/`update` is preferred for the contract-based workflow.

### Common migration gotchas
- **`@prisma/client` lags behind `prisma`**: v8-rc had no client. Pin the client to the latest stable that has a published `@prisma/client` (currently 7.10.0) and let the CLI be newer.
- **Config file is required to be ESM-ish**: Prisma 7+ reads the config via dynamic import. The file MUST have a default export. `prisma.config.ts` is fine; `prisma.config.js` works too. `prisma.config.cjs` is supported but odd.
- **Prisma 8 errors are JSON-shaped, not human-shaped**: errors come back as `{"envelope":{"ok":false,"error":{"code":"CLI.X","summary":"..."}}}`. This is intentional (programmatic error handling) but painful to read. Pipe through `| python3 -c "import json,sys;d=json.load(sys.stdin); print(d['envelope']['error']['summary'])"` for one-line summaries.
- **`prisma skills sync` shows up as a warning on every Prisma 8 command**: it's recommending you install Prisma's agent-skills. The warning is informational; ignore it unless you're building an agent.

## Health Checks & Monitoring

### Service Health Endpoints
```typescript
// apps/api/src/app/api/health/route.ts
export async function GET() {
  const db = await prisma.$queryRaw`SELECT 1`;
  const redis = await redis.ping();
  return NextResponse.json({ status: 'ok', database: !!db, redis });
}
```

### Docker Health Checks
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3001/api/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

## Common Pitfalls & Fixes

| Issue | Solution |
|-------|----------|
| Next.js SWC missing on ARM64 | Use Babel transpilation in webpack config |
| Vercel can't resolve workspace deps | Package shared as tarball, reference locally |
| Prisma client not generated in Docker | Run `npx prisma generate` in deps stage |
| Database connection refused | Add `depends_on` with `condition: service_healthy` |
| WebSocket not working behind Nginx | Add `proxy_http_version 1.1` + upgrade headers |
| Vercel build fails on workspace | Package shared as tarball, don't use `*` |
| Prisma migrate fails in prod | Use `prisma migrate deploy` not `dev` |
| Drei `shaderMaterial` fails TS with "cannot be used as a JSX component" | Cast: `const M = shaderMaterial(...) as unknown as React.ComponentType<{attach?:string;transparent?:boolean;uColor1?:THREE.Color}>;` |
| RSC page calls `prisma` directly with no DB available at build | Wrap in `try { ... } catch { return { projects: [], categories: [] } }` and use `export const dynamic = 'force-dynamic'` |
| `next lint` complains about a `load` function in `useEffect` deps | Intentional design — load is async and depends on filter state. Either wrap in `useCallback` and add to deps, or disable the rule per-file with `// eslint-disable-next-line react-hooks/exhaustive-deps` |
| `npm install` prunes dev deps (tsc, next CLI disappear) | Re-run `npm install --include=dev`. The Prisma `postinstall: prisma generate` hook is expensive but required |
| JWT in `localStorage` (XSS-vulnerable) | Use httpOnly cookies set by the server. With Next.js App Router route handlers, use `cookies().set()` from `next/headers` |
| JWT in middleware uses `jsonwebtoken` (Node-only, breaks edge runtime) | Use `jose` instead — works in edge runtime, smaller, modern API |
| Prisma 5.x on ARM64 (Termux / Raspberry Pi / AWS Graviton): `dlopen failed: libquery_engine-debian-openssl-1.1.x.so.node is for EM_X86_64` | See `references/prisma-arm64-dev.md` for the full recipe. Short version: Prisma 5's `@prisma/get-platform` reports `debian-openssl-1.1.x` regardless of CPU on Termux (Android maps to "debian"), so even shipping `linux-arm64-openssl-1.1.x.so.node` next to it doesn't help — the runtime still picks the x86_64 binary. Easiest path: upgrade to Prisma 7+ which uses the WASM Query Compiler and doesn't need the native engine. Harder path: write a Prisma-API-compatible local fallback in `lib/prisma.local.ts` that uses `pg` (Postgres) or `mariadb` (MySQL) directly, dispatching from `lib/prisma.ts` based on `process.arch === 'arm64'` or `PRISMA_FORCE_LOCAL=1`. |
| Prisma 5.x: setting `binaryTargets = ["native", "linux-arm64-openssl-1.1.x"]` doesn't fix the engine-on-ARM64 issue | The `.so.node` IS downloaded into `node_modules/.prisma/client/libquery_engine-linux-arm64-openssl-1.1.x.so.node`, but `@prisma/get-platform` still returns `debian-openssl-1.1.x` for the runtime (Termux/Android maps to "debian" regardless of actual CPU). The `dlopen` call hits the x86_64 binary, not the ARM one. Deleting the x86_64 file forces an error like "Prisma Client could not locate the Query Engine for runtime debian-openssl-1.1.x" — even though the ARM binary sits right next to it. Fix: `process.env.PRISMA_QUERY_ENGINE_BINARY` is the only knob Prisma 5 honors, but the runtime still ignores it because the engine path resolver hardcodes the platform string. Upgrading to Prisma 7 is the real fix. |
| `npm install prisma@7.x` errors with `notarget No matching version found for @prisma/client@7.x` | Prisma 7+ ships the CLI/engine in `prisma` and `@prisma/cli` packages, but `@prisma/client` follows a separate release cadence. As of 8.0.0-rc.12, `@prisma/client` latest stable is still 7.10.0. Pin both: `prisma@7.10.0` + `@prisma/client@7.10.0`, or `prisma@8.x` + `@prisma/client@7.10.0` (CLI independent of client) if the API surface matches. Mixing is fine for `prisma deploy` since the CLI doesn't load the client at all. |
| `prisma db push` succeeds but `prisma generate` produces a client that uses `runtime/library.js` which `dlopen`s the x86_64 engine anyway | The "library" engine is just a different search path — `runtime/library.js` ends in `process.dlopen(n, r, ...)` against `libquery_engine-debian-openssl-1.1.x.so.node` regardless. Adding `previewFeatures: ["driverAdapters"]` + `engineType: "library"` does NOT avoid the native engine. Only the Prisma 7+ Query Compiler or a custom `lib/prisma.local.ts` fallback does. |
| Prisma 7 schema config: `url = env("DATABASE_URL")` inside `datasource` block fails with `P1012` | Prisma 7 moved the `url` property out of `datasource`. Create `prisma.config.ts` at the project root: `import { defineConfig } from 'prisma/config'; export default defineConfig({ schema: 'prisma/schema.prisma', migrations: { path: 'prisma/migrations' }, datasource: { url: process.env.DATABASE_URL! }, experimental: { adapter: true } })`. The `datasource` block in `schema.prisma` then only contains `provider` and (optionally) `relationMode`. |
| Prisma 8 (Composer) config: `prisma db push` no longer exists | Prisma 8 replaces it with `prisma db init` (bootstrap a DB and sign it) and `prisma db update` (apply changes to existing DB). Config now requires `definePrismaConfig` from `@prisma/cli-engine` (NOT `prisma/config`) with a nested `orm: { schema, migrations, datasource }` block — top-level `schema`/`migrations`/`datasource` are rejected with `CLI.CONFIG_UNKNOWN_SECTION`. |
| Prisma 8 deploy: `bunx @prisma/cli app deploy` from v7 docs doesn't work; `prisma deploy <entry>` is the new command | `app deploy` was renamed `prisma deploy <entry>` in v8. `<entry>` is a module path (e.g. `app/prisma.compute.ts`) that exports a fetch-handler wrapping your app. Next.js 14 doesn't ship a Prisma Compute entry — you have to author one (typically a small Hono/Express adapter that pipes requests to the built Next.js handler). This is a non-trivial refactor; deferring it is a valid choice — stay on Prisma 7 and use the v7 deploy commands from the docs. |
| Prisma 8 install: `npm install` errors with peer warnings about three.js / @monogrid/gainmap-js | Prisma 8's transitive deps (e.g. `@monogrid/gainmap-js@3.4.0`) require `three@>=0.159.0` but your `@react-three/drei@9.x` pins `three@0.158.0`. The install errors out even with `--force` (npm's `ERESOLVE` aborts). Workaround: `npm install --force --ignore-scripts` (skips the Prisma postinstall that fetches the engine binary — run `prisma generate` manually after). Better: stay on Prisma 7 until three.js / drei update their peer ranges. |
| Termux `mysqld --daemonize` errors with `unknown option '--daemonize'` | The `mysqld` binary on Termux is actually `mariadbd` (renamed, with old name kept as compat shim). MariaDB 12.x removed the `--daemonize` flag. Run it in the background instead: `mysqld --user=$(whoami) --datadir=/data/.../mysql/data --socket=/data/.../mysql/mysql.sock --port=3306 --bind-address=127.0.0.1 --pid-file=/data/.../mysql/mysqld.pid --log-error=/data/.../mysql/mysqld.err &` then `disown` (or use the `terminal(background=true)` pattern). The socket file path MUST be in a directory the user owns — `/tmp` is not writable in some Termux setups. |
| Prisma 5.x `runtime/library.js` calls `process.dlopen(...)` which fails on Termux Android ARM64 with `dlopen failed: ... is for EM_X86_64 (62) instead of EM_AARCH64 (183)` | This is the entire reason `lib/prisma.local.ts` exists. The runtime loads the binary at module-init time; you can't catch it with try/catch around the API call. The fix is to make `lib/prisma.ts` lazy: export a Proxy that defers `new PrismaClient(...)` until first query, and at that point check `process.arch` and `PRISMA_FORCE_LOCAL` — if ARM, dynamic-import the local fallback. `lib/prisma.local.ts` is the only place that touches the raw `pg`/`mariadb` driver. On x86_64 production the Proxy never imports the fallback, so it's dead code that doesn't get bundled. |
| Hermes terminal tool rejects `bun` binary with `unexpected e_type: 2` | `bun` is a glibc ELF binary. Hermes's terminal wrapper rejects non-Bionic ELF executables on Android. Don't try to install bun via `curl | bash` on Termux as a workaround — it installs fine but Hermes can't exec it. On Termux, use `npx` instead of `bunx`. The Hermes wrapper error code 2 = `ET_EXEC` (normal executable format) being rejected. |
| Prisma 7 breaking change: `url` property in `datasource` block no longer accepted | Move the connection URL to `prisma.config.ts` (created at the project root): `import { defineConfig } from 'prisma/config'; export default defineConfig({ schema: 'prisma/schema.prisma', datasource: { url: process.env.DATABASE_URL! } })`. The `datasource` block in `schema.prisma` then only contains `provider` and `relationMode`. |
| `bunx` invoked on Termux fails with "unexpected e_type: 2" | `bun` is a glibc ELF binary and Hermes's terminal wrapper rejects non-Bionic ELF executables on Android. On Termux, use `npx` instead. Don't try to install bun via `curl | bash` as a workaround — it installs but Hermes can't exec it. |
| Postgres `?` placeholders in raw SQL fail with `syntax error at or near "LIMIT"` | Postgres uses `$1, $2, …` not `?`. The `pg` driver does NOT auto-rewrite. Write a tiny `pgPlaceholders(sql)` helper that increments a counter and replaces each `?` with `$N` before calling `pool.query()`. MySQL/MariaDB uses `?` natively, so make the rewrite conditional on detected DB kind. |
| Prisma `engineType: "library"` + `previewFeatures: ["driverAdapters"]` does NOT avoid the native engine | The library engine code (`runtime/library.js`) still calls `dm()` which `dlopen`s the `.so.node` file. Setting `engineType = "library"` only changes the search path, not the binary format requirement. The only real escape is Prisma 7+ with the Query Compiler, or the `lib/prisma.local.ts` fallback pattern. |

## Module Boundaries for Single Next.js Apps

When not using a monorepo, keep these concerns in separate modules so the next session can find things:

```
lib/
  prisma.ts        — PrismaClient singleton (HMR-safe)
  auth.ts          — JWT sign/verify, bcrypt hash, cookie helpers, getCurrentUser(FromRequest)
  api.ts           — NextResponse wrappers (ok, badRequest, forbidden, notFound, serverError, handleApiError)
                    — ONLY generic HTTP helpers + Zod/Prisma error mapping + small utilities (slugify, generateOrderNumber)
  pricing.ts       — Business logic constants (PROJECT_TYPES, PRICING_PLANS, calcTotalPrice)
  index.ts         — Re-exports for client code
```

**Common slip:** putting `getCurrentUserFromRequest` in `lib/api.ts` because it "feels like a request helper." It's an auth helper. Keep `lib/api.ts` dependency-light — no `jose`, no `bcrypt`, no Prisma, no `cookies()` from `next/headers`. The API routes that need auth should import from `@/lib/auth` explicitly. Mixing these creates subtle bundle issues and confuses the next reader.

## Verification Checklist

- [ ] All services have health checks
- [ ] `.env.example` exists for all apps
- [ ] No secrets in git history
- [ ] Multi-stage Docker builds use `< 500MB` images
- [ ] `output: 'standalone'` in next.config.js
- [ ] Prisma client generated in Docker build
- [ ] Database migrations run on deploy
- [ ] SSL certificates configured
- [ ] WebSocket proxy configured in Nginx
- [ ] Environment variables set in platform dashboard
- [ ] Database backups scheduled
- [ ] Log aggregation configured

## Verification on Limited Platforms (Termux / ARM64 / no Docker)

When the build environment is not a normal Linux box (e.g. Termux on Android ARM64, sandboxed CI, no Docker), `next build` will fail even though the code is correct. Do NOT try every workaround — recognize the platform limit and verify differently.

### Symptoms
- `Failed to download swc package from https://registry.npmjs.org/@next/swc-android-arm64-...`
- `TypeError: bindings.transform is not a function` after SWC falls back to WASM
- `Cannot find module 'webpack/lib/javascript/BasicEvaluatedExpression'`

### What's happening
Next.js 14 needs a native SWC binary for the host's CPU+OS. On Termux Android ARM64 the `@next/swc-android-arm64@14.x.x` package is often not published for the exact Next.js version installed. `useWasmBinary: true` only works as opt-in and the `bindings.transform` API surface differs from the native one.

### What to do instead (in priority order)

1. **Confirm TypeScript compiles cleanly**:
   ```bash
   DATABASE_URL=... npx tsc --noEmit
   ```
2. **Confirm ESLint passes**:
   ```bash
   npx next lint
   ```
3. **Confirm Prisma client generates**:
   ```bash
   DATABASE_URL=... npx prisma generate
   ```
4. **Confirm `next build` would succeed on x86_64** by:
   - Making sure `tsc --noEmit` is 0 errors (TS errors are the real blocker)
   - Reading the `next.config.js` for host-platform features that won't work on the verification host but will work on Linux (SWC wasm fallback, standalone output, etc.)
5. **Tell the user explicitly**: "Build will succeed on Hostinger KVM (x86_64); I can't verify locally because [reason]. TypeScript + ESLint + Prisma all pass." Do not pretend the build was verified when it wasn't.

### Reinstall dependency gotcha
If `node_modules/.bin/tsc` disappears mid-session, you ran something that pruned dev deps. Re-run:
```bash
npm install --include=dev
```
The Prisma `postinstall: "prisma generate"` hook makes `npm install` expensive but is required — do not disable it.

## Git Push Without Cached Credentials

When the user asks to push to GitHub but no credentials are stored (`~/.netrc` missing, no `~/.git-credentials`, no `gh` CLI installed, no `GITHUB_TOKEN` env var):

1. **Don't silently fail.** `git push` will error with `fatal: could not read Username for 'https://github.com'`.
2. **Ask via `clarify`** — give the user two clear options: paste a PAT (one-shot push) or take the commands to run locally.
3. **If they paste a token:**
   - Use it inline in the remote URL: `git remote set-url origin https://<TOKEN>@github.com/owner/repo.git`
   - Push
   - **Immediately scrub it:** `git remote set-url origin https://github.com/owner/repo.git` so the token isn't persisted in `.git/config`
   - **Warn the user the token is compromised** — it traveled in plain text and should be revoked + replaced at https://github.com/settings/tokens
4. **If they choose local commands**, give them the exact sequence:
   ```bash
   cd ~/org-portfolio
   git remote add origin https://github.com/OWNER/REPO.git
   git push -u origin develop
   ```

Do NOT store tokens in `~/.netrc` or environment variables for "later use" — that's exactly how credentials leak. One-shot inline only.