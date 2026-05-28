---
sprint: 0
fecha: 2026-05-25
tema: dependencias
---

# Sprint 0 — Packages y Stack instalado

> Versiones exactas de runtime y librerías al cerrar Sprint 0. Es la **foto base** sobre la cual se construye todo lo demás.

Sprint maestro: [[Sprint 0 - Setup (25 may)]]

---

## 🧱 Capa runtime / herramientas

| Capa | Versión | Notas |
|---|---|---|
| Node.js LTS | **22.x** (24.x dev OK) | Fijo en `engines` y `.nvmrc` |
| pnpm | **11.3.0** | Fijo en `packageManager` |
| TypeScript | **5.7+** | `strict` + `exactOptionalPropertyTypes` |
| Turborepo | **2.x** | Pipeline raíz |

## 🛰 Backend — `apps/api`

| Paquete | Versión | Para qué |
|---|---|---|
| `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express` | `^11.0.1` | Framework base |
| `@nestjs/throttler` | `^6.4.0` | Rate-limit 10/10s · 30/min · 60/h por IP |
| `express` | `~4.21.2` | Adapter HTTP (no Fastify) |
| `@trpc/server` | `^11.0.0-rc.660` | RPC tipado con `apps/web` |
| `helmet` | `^8.0.0` | Headers de seguridad |
| `pino` + `pino-http` + `pino-pretty` | `^9.5.0` / `^10.3.0` / `^13.0.0` | Logging estructurado con PII redaction |
| `ioredis` | `^5.4.2` | Cliente Redis (BullMQ + sessions + cache) |
| `zod` | `^3.24.1` | Validación de inputs |
| `reflect-metadata`, `rxjs` | — | Decoradores de Nest |

> Paquetes _agregados_ en Sprint 1 (no eran parte de Sprint 0): `jose`, `bcryptjs`, `cookie-parser`, `arctic`, `agora-token`. Documentados en [[Sprint 1 - Auth y Usuarios (26-27 may)]].

## 🧰 Workers — `apps/workers`

| Paquete | Versión | Para qué |
|---|---|---|
| `bullmq` | `^5.34.5` | Cola de jobs sobre Redis |
| `ioredis` | `^5.4.2` | Cliente Redis |
| `pino` + `pino-pretty` | `^9.5.0` | Logging |
| `@globeliv/database`, `@globeliv/zod-schemas` | workspace | Esquemas y DB compartidos |

## 🖥 Frontend — `apps/web`

| Paquete | Versión | Para qué |
|---|---|---|
| `next` | `^15.1.3` | App Router |
| `react`, `react-dom` | `^19.0.0` | React 19 |
| `@globeliv/ui` (workspace) | — | Re-exports HeroUI + tokens |
| `tailwindcss` | `^4.0.0` | Tailwind 4 (vía `@tailwindcss/postcss`) |
| `framer-motion` | `^11.15.0` | Animaciones (peer de HeroUI) |
| `lucide-react` | `^0.469.0` | Iconos |
| `@tanstack/react-query` | `^5.62.10` | Estado servidor |
| `@trpc/client` + `@trpc/react-query` | `^11.0.0-rc.660` | Hooks tipados al `AppRouter` |
| `agora-rtc-sdk-ng` | `^4.24.3` | Cliente Agora (preparado para Sprint 2) |

> Estado de cliente: **Zustand** + `react-hook-form` + `zod` — listado en CLAUDE.md como decisión pero no instalado en Sprint 0 (entran cuando aparece la primera form compleja en Sprint 1).

## 📚 Packages internos (workspace)

### `@globeliv/database`
- `drizzle-orm` `^0.38.x` + `drizzle-kit` (cli)
- Cliente PG (`postgres` driver)
- Re-exporta `schema` namespace para que `apps/api` haga `db.select().from(schema.users)`

### `@globeliv/zod-schemas`
- Solo `zod` como dependencia
- Exporta tipos compartidos: `auth`, `users`, `streams`, `jobs/*`, `trpc` (helpers)

### `@globeliv/ui`
- Re-exporta `@heroui/react` (regla: prohibido importar directo de `@heroui/react` en apps)
- `src/tokens.css` con `@theme` de Tailwind 4 — única fuente de colores/tipos/radios

### `@globeliv/config-typescript`
- `tsconfig.base.json`, `tsconfig.nest.json`, `tsconfig.next.json`, `tsconfig.lib.json`
- Apps extienden el preset apropiado

### `@globeliv/config-eslint`
- Reglas compartidas: `eslint-plugin-import`, no-default-exports (con overrides), import sorting

---

## 🎨 Design tokens — `packages/ui/src/tokens.css`

Sprint 0 dejó tokens base:

- `--color-brand-red: #ff3344` — color "Liv" y estados live
- `--color-brand-ink: #0a0a0a` — texto principal
- `--color-brand-paper: #fafafa` — fondo claro
- Tipografía: **Inter** (sans), **JetBrains Mono** (mono)
- Radios: `8 / 12 / 16 / pill`

> En Sprint 1 se evoluciona a **Kinetic v1**: `--brand-red: #ff3b4e` + `--brand-navy-*` + gradiente + fuentes `Inter + Space Grotesk`. Ver [[Sprint 1 — Profile, Settings y OAuth]].

**Regla operativa:** prohibido `#hex` o `rgb()` crudos en componentes — solo `var(--color-brand-*)` o utilities Tailwind.

---

## ⚖ Decisiones de versión / pinning

- **Next 15 + React 19**: bet temprano por App Router maduro. Riesgo asumido — algunas libs aún no soportan React 19, lo manejamos caso a caso.
- **tRPC 11 rc**: estable de facto; el major bump trae HTTP batch links mejorados.
- **Tailwind 4** (vs 3): elegido por `@theme` directives nativas. Trade-off conocido: integración con HeroUI requiere `@source` manual (deuda en [[Sprint 1 - Auth y Usuarios (26-27 may)]]).
- **NestJS 11**: alineado con Express 5 en theory, pero Express **4.21** porque 5 todavía rompe módulos comunitarios. Workaround documentado en [[Sprint 1 — Sistema de Auth]].

---

## 🔗 Referencia rápida

Archivos donde ver versiones definitivas:

- `Globeliv/package.json` — devDeps raíz
- `Globeliv/apps/api/package.json` — backend
- `Globeliv/apps/workers/package.json` — colas
- `Globeliv/apps/web/package.json` — frontend
- `Globeliv/pnpm-lock.yaml` — versiones exactas resueltas
