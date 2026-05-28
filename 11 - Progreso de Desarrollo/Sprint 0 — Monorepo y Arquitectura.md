---
sprint: 0
fecha: 2026-05-25
tema: monorepo
---

# Sprint 0 — Monorepo y Arquitectura

> Estructura física del repo `Globeliv/`. Cómo se organizan apps, packages y la build pipeline.

Sprint maestro: [[Sprint 0 - Setup (25 may)]]

---

## 🗂 Layout final del repo

```
Globeliv/
├── apps/
│   ├── web/        Next.js 15 App Router — globeliv.com
│   ├── api/        NestJS 11 + Express + tRPC — api.globeliv.com
│   └── workers/    BullMQ + Redis — proceso separado (Railway)
│
├── packages/
│   ├── config-typescript/   tsconfig bases (nest, next, lib)
│   ├── config-eslint/       reglas ESLint compartidas
│   ├── database/            Drizzle schema + migrations + client
│   ├── zod-schemas/         contratos compartidos (auth, users, streams, jobs)
│   └── ui/                  tokens.css + re-exports HeroUI
│
├── .agents/        nestjs-doctor (auditor backend)
├── .claude/        skills instalados a nivel proyecto
├── .github/        workflows CI
│
├── CLAUDE.md           ← fuente de verdad operativa
├── ROADMAP.md          ← pendientes por sprint
├── turbo.json          ← pipeline tasks (build, dev, lint, typecheck)
├── pnpm-workspace.yaml ← apps/* + packages/*
└── package.json        ← scripts raíz que delegan a turbo
```

---

## ⚙️ Turborepo — pipeline

`turbo.json` define los tasks. Lo importante:

- `build` depende de `^build` (build deps primero) → orden topológico automático
- `dev` es `cache: false` y `persistent: true` para watchers
- `lint`, `typecheck`, `test` también ordenadas topológicamente
- `@globeliv/database#db:generate` y `#db:migrate` son tasks específicas no cacheables

Caché remoto: no configurado aún (Turbo Cloud / Vercel Remote Cache pendiente).

## 📦 pnpm-workspace.yaml

```yaml
packages:
  - "apps/*"
  - "packages/*"

allowBuilds:
  "@heroui/shared-utils": true
  "@nestjs/core": true
  esbuild: true
  msgpackr-extract: true
  sharp: false
  unrs-resolver: false
```

La lista `allowBuilds` es **explícita** (pnpm 11 lo exige). Cada postinstall que se permite está justificado:
- `@heroui/shared-utils`, `@nestjs/core` → necesitan `reflect-metadata` setup
- `esbuild`, `msgpackr-extract` → binarios nativos
- `sharp: false` → no lo usamos por ahora; cuando entren thumbnails de R2 se activará
- `unrs-resolver: false` → resolver experimental, lo dejamos apagado

---

## 🧱 Convenciones del repo

Vienen de `CLAUDE.md` §8:

- **Archivos** en `kebab-case.ts`. Componentes React en `PascalCase.tsx`.
- **Funciones < 50 líneas. Archivos < 400 líneas.**
- **Comentarios en español natural explicando *por qué*.** Nunca el *qué*.
- **No** `default exports` salvo cuando un framework lo exija (`page.tsx`, `layout.tsx` de Next).
- **Imports ordenados:** builtin → externos → `@globeliv/*` → relativos. Enforced por `eslint-plugin-import` (configurado en `@globeliv/config-eslint`).
- **Conventional Commits** con `Co-Authored-By: Claude Opus 4.7` cuando aplica.

---

## 🛣 Visual de la arquitectura desplegable

```
┌─────────── Vercel ────────────────────┐
│  apps/web → globeliv.com              │
│  apps/admin → admin.globeliv.com (S7) │
└────────────────┬──────────────────────┘
                 │ tRPC + HTTP (tipos compartidos)
                 ▼
┌─────────── Railway ────────────────────────┐
│  apps/api  (NestJS sobre Express)          │
│  ├── /webhooks/*  HMAC guard → cola        │
│  ├── /trpc/*      AppRouter completo       │
│  ├── /health      DB + Redis check         │
│  └── ThrottlerGuard 10/10s · 30/min · 60/h │
│                                            │
│  apps/workers  (BullMQ proceso separado)   │
│  ├── inbound-dispatcher                    │
│  ├── process-*  lógica core por dominio    │
│  └── send-notification                     │
│                                            │
│  Postgres 17  ← migrations versionadas     │
│  Redis        ← BullMQ + sessions + pub/sub│
└─────────────────────────────────────────────┘
                 │
                 ▼
   Cloudflare R2 · Agora · Stripe Connect · FCM
```

---

## 🔗 Sub-notas relacionadas

- [[Sprint 0 — Packages y Stack]]
- [[Sprint 0 — Skills instalados]]
- [[Sprint 0 — DB y Migraciones]]
