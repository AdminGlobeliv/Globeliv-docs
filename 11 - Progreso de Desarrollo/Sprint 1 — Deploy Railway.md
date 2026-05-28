---
sprint: 1
fecha_inicio: 2026-05-27
fecha_fin: 2026-05-27
tema: deploy-backend
---

# Sprint 1 — Deploy Railway (`apps/api` → api.globeliv.com)

> Backend NestJS desplegado en Railway. **Pivot** Railpack → Dockerfile multi-stage tras crash del parser. Postgres y Redis productivos también en Railway.

Sprint maestro: [[Sprint 1 - Auth y Usuarios (26-27 may)]]

---

## 🎯 Lo que se quería

Backend `apps/api` corriendo en producción con dominio propio (`api.globeliv.com`), Postgres y Redis gestionados, auto-deploy desde `main`.

## 🛣 Camino que se siguió (3 intentos)

### Intento 1 — `railpack.json` completo (`d60fc73`)

```json
{
  "provider": "node",
  "packages": { "node": "22", "pnpm": "11.3.0" },
  "steps": {
    "install": { ... },
    "build": { ... },
    "caches": { ... }
  },
  "deploy": { ... }
}
```

❌ Railpack v0.23 crasheó por features experimentales (`caches.type`, `inputs.step` refs).

### Intento 2 — `railpack.json` mínimo (`c9dbf29`)

Solo `provider / packages / steps(install,build) / deploy`. **El parser pasaba pero el build no generaba logs visibles** — imposible debuggear.

### Intento 3 — **Dockerfile multi-stage** (`549c89e`) ✅

Pivot a Dockerfile explícito. Railway lo detecta automáticamente y los logs del build son visibles.

---

## 🐳 Dockerfile final

Archivo: `Globeliv/Dockerfile`

3 stages:

```dockerfile
# Stage 1: deps
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@11.3.0 --activate

FROM base AS deps
# Copia manifests primero → cache hit cuando solo cambia código
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml .npmrc ./
COPY apps/api/package.json ./apps/api/
COPY apps/web/package.json ./apps/web/
COPY apps/workers/package.json ./apps/workers/
COPY packages/*/package.json ./packages/*/  # (un COPY por package)
RUN pnpm install --frozen-lockfile

# Stage 2: build sólo @globeliv/api... (api + workspace deps)
FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm --filter @globeliv/api... build

# Stage 3: runtime mínimo
FROM node:22-alpine AS runtime
ENV NODE_ENV=production
COPY --from=build /app/apps/api/dist ./apps/api/dist
COPY --from=build /app/apps/api/node_modules ./apps/api/node_modules
EXPOSE 4000
CMD ["node", "apps/api/dist/main.js"]
```

**Por qué multi-stage:**
- Build artifacts no viajan a runtime → imagen final ~150 MB
- `tsup` bundlea los `@globeliv/*` internos → `dist/main.js` es self-contained
- Solo se copian `node_modules` de runtime (Nest, Express, etc.), no devDeps

### `.dockerignore` (`549c89e`)

```
node_modules/
**/node_modules/
.next/
**/dist/
.turbo/
.env
.env.*
GlobeLiv-docs/
*.md
.git/
.github/
```

---

## 🔧 Env vars críticos en Railway

| Variable | Valor en prod | Notas |
|---|---|---|
| `NODE_ENV` | `production` | activa cookies `secure: true`, CORS strict |
| `PORT` | (auto-injectado por Railway) | El `env.ts` hace fallback `API_PORT ?? PORT` |
| `API_HOST` | `0.0.0.0` | listen all interfaces |
| `DATABASE_URL` | `postgresql://...railway.internal:5432/...` | proxy interno Railway |
| `REDIS_URL` | `redis://default:...railway.internal:6379` | mismo network privado |
| `JWT_SECRET` | (rotable, 32+ chars) | generado con `openssl rand -base64 32` |
| `JWT_ISSUER` | `globeliv.com` | matchear con verify |
| `JWT_AUDIENCE` | `globeliv-web` | matchear con verify |
| `WEB_PUBLIC_URL` | `https://www.globeliv.com` | usado por CORS allowlist + redirect OAuth |
| `GOOGLE_OAUTH_CLIENT_ID` / `_SECRET` | de Google Cloud Console | redirect URI `https://api.globeliv.com/oauth/google/callback` |

> `.env.example` en el repo tiene la lista completa con notas. `.env` (real) NO está en git.

---

## 🌐 Subdominio `api.globeliv.com`

Configurado en Railway > Settings > Custom Domains:

- CNAME `api.globeliv.com` → `<railway-domain>.up.railway.app`
- TLS auto-managed por Railway

**Por qué subdominio y no path-based proxy:**

- Mejor separación operativa (frontend Vercel, backend Railway)
- Permite **cookies same-site** (eTLD+1 compartido = `globeliv.com`)
- Sin necesidad de `next.config.ts` rewrites complejos

> Esta decisión arregló dos bugs de cookies — ver [[Sprint 1 — CORS y Cookies cross-domain]].

---

## 🏥 Health checks

Endpoint: `GET https://api.globeliv.com/health`

Implementado en `Globeliv/apps/api/src/modules/health/health.controller.ts`. Prueba:
- Conexión a Postgres (`SELECT 1`)
- Conexión a Redis (`PING`)

Si cualquiera falla → 503. Railway lo usa para decidir si la instancia está lista para tráfico.

---

## 🔄 Auto-deploy

Railway detecta push a `main` → rebuilda imagen → swap con zero-downtime (rolling).

**No hay GitHub Actions workflow** para Railway (lo gestiona el servicio directamente). El gate es el CI workflow que valida lint/typecheck/build **antes** de mergear a `main`.

---

## 📂 Archivos relevantes

- `Globeliv/Dockerfile`
- `Globeliv/.dockerignore`
- `Globeliv/apps/api/src/env.ts` — Zod validation de env vars
- `Globeliv/apps/api/src/main.ts` — bootstrap NestJS + express
- `Globeliv/.env.example` — plantilla con todas las vars
