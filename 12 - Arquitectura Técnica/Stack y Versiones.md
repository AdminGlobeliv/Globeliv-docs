---
sección: arquitectura
tema: stack
---

# Stack y Versiones

> Snapshot del estado actual. **Atemporal** — se actualiza cuando algo cambia. Si quieres ver "qué se instaló cuándo" → [[Sprint 0 — Packages y Stack]].

---

## 🏗 Runtime y herramientas

| Capa | Tecnología | Versión |
|---|---|---|
| Runtime | Node.js LTS | **22.x** (24.x dev OK) |
| Package manager | pnpm | **11.3.0** |
| Lenguaje | TypeScript | **5.7+** (strict + exactOptionalPropertyTypes) |
| Build orquestador | Turborepo | **2.x** |
| Build apps backend | tsup | 8.x |
| Build app frontend | Next compiler | 15.x |

## 🖥 Frontend — `apps/web`

| Paquete | Versión | Para qué |
|---|---|---|
| `next` | ^15.1.3 | App Router |
| `react`, `react-dom` | ^19.0.0 | React 19 |
| `tailwindcss` | ^4.0.0 | Tailwind 4 (`@theme`) |
| `@tanstack/react-query` | ^5.62.10 | Estado servidor |
| `@trpc/client`, `@trpc/react-query` | ^11.0.0-rc | Hooks tipados al AppRouter |
| `framer-motion` | ^11.15.0 | Animaciones (peer de HeroUI) |
| `lucide-react` | ^0.469.0 | Iconos |
| `agora-rtc-sdk-ng` | ^4.24.3 | Cliente Agora |
| `@globeliv/ui` (workspace) | — | Re-exports HeroUI + tokens |

## 🛰 Backend API — `apps/api`

| Paquete | Versión | Para qué |
|---|---|---|
| `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express` | ^11.0.1 | Framework |
| `@nestjs/throttler` | ^6.4.0 | Rate-limit 10/10s · 30/min · 60/h |
| `express` | ~4.21.2 | Adapter HTTP |
| `@trpc/server` | ^11.0.0-rc | RPC tipado |
| `helmet` | ^8.0.0 | Headers de seguridad |
| `pino` + `pino-http` | ^9.5.0 / ^10.3.0 | Logging con PII redaction |
| `ioredis` | ^5.4.2 | Cliente Redis |
| `zod` | ^3.24.1 | Validación de inputs |
| `jose` | ^6.2.3 | JWT (ESM nativo) |
| `bcryptjs` | ^3.0.3 | Password hashing |
| `cookie-parser` | ^1.4.7 | Parse cookies |
| `arctic` | ^3.7.0 | OAuth Google (PKCE) |
| `agora-token` | ^2.0.5 | Firma tokens Agora |

## 🧰 Workers — `apps/workers`

| Paquete | Versión | Para qué |
|---|---|---|
| `bullmq` | ^5.34.5 | Cola sobre Redis |
| `ioredis` | ^5.4.2 | Cliente Redis |
| `pino` | ^9.5.0 | Logging |
| `zod` | ^3.24.1 | Validación de jobs |

## 📚 Packages workspace (`packages/`)

| Package | Función |
|---|---|
| `@globeliv/database` | Drizzle schema + migrations + client (postgres.js driver) |
| `@globeliv/zod-schemas` | Contratos compartidos (auth, users, streams, jobs) |
| `@globeliv/ui` | Re-exports HeroUI + design tokens (`tokens.css`) |
| `@globeliv/config-typescript` | tsconfig bases (nest, next, lib) |
| `@globeliv/config-eslint` | Reglas ESLint compartidas |

---

## 🗄 Datos y persistencia

| Capa | Tecnología | Versión |
|---|---|---|
| ORM | Drizzle | 0.38.x |
| DB | PostgreSQL | **17** (18.3 dev local OK) |
| Cache / queue | Redis | latest (Railway managed) |
| Colas | BullMQ | 5.x |
| Storage archivos | Cloudflare R2 | — (diferido al activar billing) |

## 🌐 Servicios externos

| Servicio | Para qué | Sprint donde entra |
|---|---|---|
| Vercel | Hosting `apps/web` | Sprint 0 |
| Railway | Hosting `apps/api` + workers + DBs | Sprint 1 |
| Google OAuth | Sign-in con Google | Sprint 1 |
| Agora | RTC video bidireccional | Sprint 2 |
| Stripe Connect | Pagos 90/10 + payouts | Sprint 4 |
| Cloudflare R2 | Archivos (avatares, thumbnails) | TBD |
| Firebase FCM | Push notifications | Sprint 6 |

---

## 🚦 Decisiones de versión / pinning destacadas

- **Next 15 + React 19** — bet temprano por App Router maduro. Riesgo: algunas libs no soportan React 19, manejamos caso a caso.
- **tRPC 11 rc** — estable de facto; HTTP batch links mejorados.
- **Tailwind 4** — elegido por `@theme` directives nativas. Trade-off: integración con HeroUI requiere `@source` manual (deuda técnica en [[Sprint 1 — Profile, Settings y OAuth]]).
- **NestJS 11 + Express 4** (no 5) — Express 5 rompe módulos comunitarios. Workaround en [[Flujo Backend (NestJS)]].
- **`agora-token` (CJS-only)** — requiere `import agoraToken from 'agora-token'` + desestructurar. ESM no funciona directo.

---

## 🔄 Cómo actualizar versiones

```bash
# Ver desactualizados
pnpm outdated

# Actualizar selectivamente (NUNCA `pnpm update` en automático)
pnpm --filter @globeliv/api up <paquete>@<version>

# Validar antes de commit
pnpm typecheck && pnpm lint && pnpm build
```

**Regla:** una versión nunca se sube sin que `pnpm build && pnpm typecheck` pase verde local + en CI.

---

## 🔗 Referencias

- `Globeliv/package.json` — devDeps raíz
- `Globeliv/apps/*/package.json` — deps por app
- `Globeliv/pnpm-lock.yaml` — versiones exactas resueltas
- `Globeliv/CLAUDE.md` §3 — tabla maestra de stack
