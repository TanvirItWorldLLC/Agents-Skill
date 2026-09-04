# Prisma on ARM64 / Termux / limited Linux hosts

Prisma 5.x and earlier ship the query engine as a native ELF binary. On most CI / VPS targets (x86_64 Linux, macOS, Windows) this works without thought. On ARM64 hosts (Termux Android, Raspberry Pi, AWS Graviton) it can break in several places — and the failure mode depends on the Prisma version.

## What's actually happening

Prisma's `node_modules/@prisma/engines/` package contains a single binary per target. As of Prisma 5.22:

```
node_modules/@prisma/engines/
├── libquery_engine-debian-openssl-1.1.x.so.node   # x86_64 only
├── schema-engine-debian-openssl-1.1.x              # x86_64 only
```

The `.so.node` file is loaded via `process.dlopen()` at module-init time inside `node_modules/@prisma/client/runtime/library.js`. If the binary's architecture doesn't match the host CPU, dlopen fails with `dlopen failed: ... is for EM_X86_64 (62) instead of EM_AARCH64 (183)`.

You can ask Prisma to ship additional targets via `binaryTargets`:

```prisma
generator client {
  provider      = "prisma-client-js"
  binaryTargets = ["native", "linux-arm64-openssl-1.1.x"]
}
```

`prisma generate` will then download the ARM64 binary into `node_modules/.prisma/client/libquery_engine-linux-arm64-openssl-1.1.x.so.node`. **But Prisma 5's runtime still picks the debian-openssl-1.1.x binary** because `@prisma/get-platform` returns `debian-openssl-1.1.x` for *all* Linux targets (Termux/Android maps to "debian" regardless of CPU). The dlopen hits the x86_64 binary, fails, and you get a misleading "engines not compatible with your system" error.

**Verifying this:**

```bash
$ file node_modules/.prisma/client/libquery_engine-*.so.node
libquery_engine-debian-openssl-1.1.x.so.node:        ELF 64-bit LSB x86-64, ...
libquery_engine-linux-arm64-openssl-1.1.x.so.node:  ELF 64-bit LSB arm64, ...
$ ./node_modules/@prisma/engines/schema-engine-debian-openssl-1.1.x --version
error: ... is for EM_X86_64 (62) instead of EM_AARCH64 (183)
```

`PRISMA_QUERY_ENGINE_BINARY` env var is supposed to override the path, but Prisma 5's engine resolver hardcodes the platform string and the override only works for matching targets.

## Three ways to unblock, in order of preference

### Option 1 — Upgrade to Prisma 7+ (the real fix)

Prisma 7.10.0 enables the **Query Compiler** (WASM) by default. The native `.so.node` is no longer required at runtime — queries compile to a portable WASM module. ARM64 hosts work out of the box.

```bash
npm install prisma@7.10.0 @prisma/client@7.10.0 --save-exact
```

The migration cost is small:
1. Remove the `url = env("DATABASE_URL")` line from the `datasource` block in `schema.prisma`.
2. Create `prisma.config.ts` at the project root — see the SKILL.md "Prisma Version Migration" section for the exact contents.
3. Update `package.json` scripts (the `prisma generate` step is unchanged, but consider `postinstall: "prisma generate || true"` defensively).
4. Run `npx prisma generate` once. The generated client lives at `node_modules/.prisma/client/default.js` (same as before).
5. Delete the `lib/prisma.local.ts` fallback if you had one — it's no longer needed.

**Verify it works on the ARM64 host:**

```bash
# These should all succeed on the ARM64 host
DATABASE_URL=... npx prisma generate
DATABASE_URL=... npx tsx -e "import { prisma } from './lib/prisma'; prisma.user.count().then(console.log).catch(console.error)"
```

### Option 2 — `lib/prisma.local.ts` fallback (when you can't upgrade)

Build a Prisma-API-compatible client using raw `pg` (Postgres) or `mariadb` (MySQL) drivers, dispatching from `lib/prisma.ts` based on `process.arch`. The local fallback is loaded dynamically on ARM64 only — x86_64 production never imports it.

**Architecture:**

```
lib/
  prisma.ts         # Public API: lazy Proxy that dispatches to Prisma or local
  prisma.local.ts   # ARM64 fallback: pg/mariadb-based Prisma-compatible surface
```

`lib/prisma.ts` pattern (Prisma 5/6 compatible, x86_64 production):

```ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

function shouldUseLocalFallback(): boolean {
  if (process.arch !== 'arm64') return false;
  // Check whether the ARM engine binary is actually present
  try {
    const arm = require.resolve('.prisma/client/libquery_engine-linux-arm64-openssl-1.1.x.so.node');
    const x64 = require.resolve('.prisma/client/libquery_engine-debian-openssl-1.1.x.so.node');
    if (fs.existsSync(arm) && !fs.existsSync(x64)) return false;  // ARM-only: try Prisma
    if (!fs.existsSync(arm) && fs.existsSync(x64)) return true;   // x64 only: fall back
    return process.env.PRISMA_LOCAL_FALLBACK === '1';
  } catch { return false; }
}

async function createClient(): Promise<PrismaClient> {
  if (!shouldUseLocalFallback()) {
    return new PrismaClient({ log: ['error', 'warn'] });
  }
  const mod = await import('./prisma.local');
  return mod.prisma as unknown as PrismaClient;
}

let clientPromise: Promise<PrismaClient> | null = null;
async function getClient(): Promise<PrismaClient> {
  if (!clientPromise) clientPromise = createClient();
  return clientPromise;
}

export const prisma: PrismaClient = new Proxy({} as PrismaClient, {
  get(_t, prop) {
    return new Proxy({} as Record<string, unknown>, {
      get(_t2, sub) {
        return async (...args: unknown[]) => {
          const c = await getClient();
          const m = (c as any)[prop];
          if (m && typeof m === 'object') {
            const f = m[sub];
            if (typeof f === 'function') return f.apply(m, args);
            return m[sub];
          }
          return undefined;
        };
      },
    });
  },
}) as PrismaClient;
```

`lib/prisma.local.ts` — minimal Prisma-compatible surface (one of each model is enough for most apps):

```ts
import { Pool as PgPool } from 'pg';
import mariadb from 'mariadb';

let pgPool: PgPool | null = null;
let mysqlPool: mariadb.Pool | null = null;
let dbKind: 'postgres' | 'mysql' | null = null;

function getPgPool(): PgPool {
  if (pgPool) return pgPool;
  const url = process.env.DATABASE_URL!;
  pgPool = new PgPool({
    connectionString: url,
    ssl: url.includes('sslmode=require') ? { rejectUnauthorized: false } : false,
    max: 5,
  });
  return pgPool;
}
function getMysqlPool(): mariadb.Pool {
  if (mysqlPool) return mysqlPool;
  const u = new URL(process.env.DATABASE_URL!);
  mysqlPool = mariadb.createPool({
    host: u.hostname, port: u.port ? Number(u.port) : 3306,
    user: decodeURIComponent(u.username), password: decodeURIComponent(u.password),
    database: u.pathname.replace(/^\//, ''), connectionLimit: 5, dateStrings: true,
  });
  return mysqlPool;
}

// Postgres uses $1, $2, ... placeholders. MySQL/MariaDB uses ?.
// The pg driver does NOT auto-rewrite, so we have to.
function pgPlaceholders(sql: string): string {
  if (dbKind !== 'postgres') return sql;
  let i = 0;
  return sql.replace(/\?/g, () => `$${++i}`);
}

async function query<T = unknown>(sql: string, args: unknown[] = []): Promise<T[]> {
  if (!dbKind) dbKind = process.env.DATABASE_URL!.startsWith('postgres') ? 'postgres' : 'mysql';
  const finalSql = pgPlaceholders(sql);
  if (dbKind === 'postgres') return (await getPgPool().query(finalSql, args)).rows as T[];
  const conn = await getMysqlPool().getConnection();
  try { return (await conn.query(sql, args)) as T[]; } finally { conn.release(); }
}
async function exec(sql: string, args: unknown[] = []): Promise<void> {
  if (!dbKind) dbKind = process.env.DATABASE_URL!.startsWith('postgres') ? 'postgres' : 'mysql';
  if (dbKind === 'postgres') await getPgPool().query(pgPlaceholders(sql), args);
  else { const c = await getMysqlPool().getConnection(); try { await c.query(sql, args); } finally { c.release(); } }
}

function cuid() { return require('node:crypto').randomUUID().replace(/-/g, '').slice(0, 25); }
function parseJSON<T>(s: unknown): T | null { if (s == null) return null; if (typeof s === 'object') return s as T; try { return JSON.parse(s as string) as T; } catch { return null; } }

// Implement the subset of Prisma methods the app actually uses:
// user.{findUnique,findMany,count,create,upsert,update,delete}
// category.{findMany,findUnique,upsert,create,update,delete}
// project.{findMany,findUnique,count,upsert,update,delete}
// order.{findMany,findUnique,count,create,update,delete}
// setting.{upsert}
//
// For each, translate Prisma semantics (camelCase → snake_case, `where: { id }` vs `id`, etc.)
// to raw SQL. The shape returned to callers must match what the production
// `prisma.user.findUnique()` etc. would return: lowercased role/status, parsed JSON,
// joined `category` object, etc.
```

**Key implementation details:**

- `role` and `status` come back from Postgres as `'ADMIN'`/`'ACTIVE'`. The app's `auth.ts` checks `role === 'ADMIN'` lowercase. Always normalize: `role: r.role.toLowerCase()`.
- `gallery`, `tags`, `technologies`, `challenges`, `solutions`, `results`, `projectDetails`, `contactInfo` are JSON columns. In Postgres they're returned as objects; in MariaDB they're returned as strings. Always parse: `parseJSONArray<T>(r.gallery)`.
- `setting.findUnique`/`upsert` has a synthetic primary key `'default'`. Both `findUnique({ where: { id: 'default' } })` and `upsert({ where: { id: 'default' }, update, create })` work against the singleton.
- For Postgres JSON columns, use `?::jsonb` cast in INSERT/UPDATE (e.g. `INSERT INTO projects (..., technologies) VALUES (?, ?::jsonb, ...)`). For MariaDB, the column type `JSON` already handles string→JSON conversion when the value is passed as a string.
- `prisma.project.findMany({ where: { status: { in: [...] } } })` — translate to `WHERE status IN (?, ?, ?)` with positional placeholders.
- `prisma.project.findMany({ orderBy: [{ featured: 'desc' }, { createdAt: 'desc' }] })` — translate to `ORDER BY featured DESC, "createdAt" DESC`.
- For joins (`project` with `category`): `SELECT p.*, c.id AS category_id_j, c.name AS category_name, ... FROM projects p LEFT JOIN categories c ON p."categoryId" = c.id`. Map the joined columns into a `category: { id, name, slug, color, icon }` object in the result.

**Calling convention (don't break it):** the public `prisma` in `lib/prisma.ts` is a Proxy. Callers do `await prisma.user.findUnique({ where: { email } })` and it works on both Prisma and local fallback. The Proxy catches `prisma.user` (returns a sub-proxy), `.findUnique` (returns an async function), and forwards to the real client.

**What this fallback does NOT support:**

- `prisma.$queryRaw` and `prisma.$executeRaw` — implement a thin wrapper if you need them; otherwise avoid in the app.
- `prisma.$transaction([...])` — wrap with a single connection if needed.
- `prisma.$connect()` / `prisma.$disconnect()` — no-op.
- Complex relational filters like `where: { category: { slug: 'web' } }` — write a JOIN instead.

### Option 3 — Just don't run the Prisma CLI on ARM64

If the code compiles cleanly with `tsc --noEmit` and the smoke test against the local fallback passes, you can ship to production (Hostinger KVM x86_64) without ever running the Prisma CLI on the ARM64 host. The CI / production host will run `prisma generate` and `prisma db push` against a real Linux box. The local fallback in `lib/prisma.local.ts` is purely for local dev.

## Recipes

### Initialize a fresh MariaDB on Termux for local dev

`mysqld` on Termux is actually `mariadbd` (renamed in MariaDB 12.x; the `mysqld` binary is a compat shim). `--daemonize` was removed. Run it in the background:

```bash
# One-time setup
mkdir -p ~/mysql/data ~/mysql/tmp
mysql_install_db --datadir=~/mysql/data --auth-root-authentication-method=normal

# Start the daemon (must use absolute paths so /tmp doesn't get picked)
mysqld \
  --user=$(whoami) \
  --datadir=/data/data/com.termux/files/home/mysql/data \
  --socket=/data/data/com.termux/files/home/mysql/mysql.sock \
  --port=3306 \
  --bind-address=127.0.0.1 \
  --pid-file=/data/data/com.termux/files/home/mysql/mysqld.pid \
  --log-error=/data/data/com.termux/files/home/mysql/mysqld.err &

# Wait 4s, then create the database + user
sleep 4
mysql --socket=/data/data/com.termux/files/home/mysql/mysql.sock -uroot <<'SQL'
CREATE DATABASE IF NOT EXISTS org_portfolio CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'portfolio'@'127.0.0.1' IDENTIFIED BY 'portfolio';
GRANT ALL PRIVILEGES ON org_portfolio.* TO 'portfolio'@'127.0.0.1';
FLUSH PRIVILEGES;
SQL
```

`/tmp` is not writable in some Termux setups, so all paths (data, socket, pid, log) must be under a directory the user owns. Use `127.0.0.1` (not `localhost`) in `DATABASE_URL` — some Termux MariaDB builds resolve `localhost` to a Unix socket that you don't have.

### Apply Prisma schema via raw SQL when `prisma db push` won't run

`prisma db push` requires the schema engine binary. On Termux ARM64, that binary is x86_64 only and `dlopen` fails. Write the schema as raw SQL and apply directly:

```sql
-- Example: MySQL schema
CREATE TABLE users (
  id VARCHAR(30) PRIMARY KEY, name VARCHAR(191) NOT NULL, email VARCHAR(191) NOT NULL UNIQUE,
  password VARCHAR(191) NOT NULL, role ENUM('ADMIN','USER') NOT NULL DEFAULT 'USER',
  status ENUM('ACTIVE','INACTIVE') NOT NULL DEFAULT 'ACTIVE', avatar VARCHAR(191),
  lastLogin DATETIME(3), createdAt DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updatedAt DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  INDEX users_email_idx (email), INDEX users_role_idx (role), INDEX users_status_idx (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
-- ... etc for categories, projects, orders, settings
```

For Postgres:

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL UNIQUE,
  password TEXT NOT NULL, role TEXT NOT NULL DEFAULT 'USER' CHECK (role IN ('ADMIN','USER')),
  status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE','INACTIVE')),
  avatar TEXT, "lastLogin" TIMESTAMPTZ,
  "createdAt" TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  "updatedAt" TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_role_idx ON users(role);
-- ... etc
```

Apply with the raw driver (`mysql` CLI or `pg.Pool` in a one-shot script). When the production host picks up the repo and runs `prisma db push`, it'll see the schema already matches and no-op.

### Smoke test the data layer end-to-end (no Next.js, no SWC)

When `next dev` and `next build` are blocked by missing ARM64 SWC binaries, exercise the data layer with a smoke test:

```ts
// scripts/smoke-db.ts
import { prisma } from '../lib/prisma';
import bcrypt from 'bcryptjs';

let passed = 0, failed = 0;
async function check(name: string, fn: () => Promise<unknown>) {
  try { await fn(); console.log(`  ✓ ${name}`); passed++; }
  catch (e) { console.log(`  ✗ ${name}\n      ${(e as Error).message}`); failed++; }
}

async function main() {
  // 1. Counts
  await check('count users', async () => {
    const n = await prisma.user.count();
    if (n < 1) throw new Error('no users');
  });
  // 2. Auth
  await check('user.findUnique by email', async () => {
    const u = await prisma.user.findUnique({ where: { email: 'admin@orgportfolio.com' } });
    if (!u) throw new Error('admin not found');
  });
  // 3. Public portfolio
  await check('project.findMany PUBLISHED', async () => {
    const ps = await prisma.project.findMany({ where: { status: 'PUBLISHED' }, take: 50 });
    if (ps.length < 3) throw new Error(`only ${ps.length}`);
  });
  // 4. Order create + lifecycle
  await check('order.create', async () => {
    await prisma.order.create({
      data: {
        orderNumber: `ORD-TEST-${Date.now()}`,
        projectType: 'portfolio', plan: 'professional', totalPrice: 7500,
        projectDetails: { title: 't', description: 'd' },
        contactInfo: { name: 't', email: 't@example.com' },
      } as never,
    } as never);
  });
  console.log(`\n${passed} passed, ${failed} failed`);
  process.exit(failed === 0 ? 0 : 1);
}
main().catch((e) => { console.error('FATAL', e); process.exit(1); });
```

Run: `DATABASE_URL=... PRISMA_FORCE_LOCAL=1 tsx scripts/smoke-db.ts`. A 22-test version of this caught every Prisma/local-fallback bug in the rebuild this came from. If the smoke test passes, the data layer is correct end-to-end even when the Next.js renderer can't compile locally.

## Verifying what the runtime will do

`@prisma/get-platform` reports the runtime's expected target:

```ts
require('@prisma/get-platform')
  // or in tsx:
  // import('@prisma/get-platform').then(m => console.log(m))
```

On Termux ARM64 this reports `debian-openssl-1.1.x` — which is why the engine resolver always picks the x86_64 binary regardless of what `binaryTargets` says. Until Prisma 7's Query Compiler became the default, this was a hard wall on ARM64 dev.

## Summary

- **First choice:** upgrade to Prisma 7.10.0. The WASM Query Compiler runs on any host.
- **Second choice:** ship a `lib/prisma.local.ts` fallback and dispatch from `lib/prisma.ts` based on `process.arch === 'arm64'`. The fallback is dead code on x86_64 production.
- **Third choice:** don't run the Prisma CLI on the ARM64 host at all. Verify with `tsc --noEmit` and a `scripts/smoke-db.ts` that exercises every `prisma.*` call. The CI / production host will run `prisma generate` and `prisma db push` itself.

For the specific build error: `dlopen failed: libquery_engine-debian-openssl-1.1.x.so.node is for EM_X86_64 (62) instead of EM_AARCH64 (183)` — only Prisma 7+ fixes this without manual intervention. Everything else is a workaround.
