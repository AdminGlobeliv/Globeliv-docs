---
sección: arquitectura
tema: adrs
---

# Decisiones de Arquitectura (ADRs)

> Las decisiones que **no se discuten salvo evidencia fuerte**. Sirven como contrato del proyecto: si llega una propuesta que rompe una ADR, debe traer datos suficientes para justificar el pivot.

> Las versiones largas viven en `Globeliv/CLAUDE.md` §10 + `ARCHITECTURE_PROMPT.md`. Esta nota resume cada ADR con su consecuencia práctica.

---

## 📋 ADRs activas

| # | Decisión | Estado | Consecuencia |
|---|---|---|---|
| 1 | **Postgres** como BD principal | ✅ activa | Sin NoSQL principal. JSON cuando aplique vía `jsonb` |
| 2 | **Drizzle** como ORM | ✅ activa | Sin Prisma. Migrations versionadas en git |
| 3 | **NestJS** para la API | ✅ activa | Sin Express crudo, sin Fastify standalone |
| 4 | **Next.js App Router** | ✅ activa | Sin Pages Router. RSC + Client mixto |
| 5 | **Cliente directo de APIs externas** | ✅ activa | Sin intermediarios SaaS (Zapier, Make) |
| 6 | **Cloudflare R2** para archivos | ✅ activa (diferida) | Sin S3. Activa cuando haya billing |
| 7 | **Monolito modular** | ✅ activa | Sin microservicios hasta 10K+ usuarios |
| 8 | **tRPC** entre Next y Nest | ✅ activa | Sin GraphQL, sin REST manual (salvo OAuth + webhooks) |
| 9 | **Drizzle migrations** en git | ✅ activa | Nunca cambios manuales en prod |
| 10 | **Design tokens** obligatorios | ✅ activa | Sin `#hex` directos en componentes |

---

## 📑 ADRs detalladas

### ADR-1: Postgres como BD principal

**Estado:** activa  
**Fecha:** 2026-05-25 (Sprint 0)

**Contexto:** GlobeLiv requiere consistencia transaccional (pagos, escrow), queries complejas (búsqueda geo, joins streamers↔streams) y data integrity (multi-tenant, soft-deletes).

**Decisión:** Postgres 17 managed (Railway). Sin Mongo, sin DynamoDB.

**Consecuencias:**
- ✅ ACID transaccional para pagos y misiones
- ✅ `jsonb` cuando algo realmente sea schema-less
- ✅ Indexes funcionales, partial unique, GIN/GiST disponibles
- ⚠️ Costo de escalar verticalmente más alto que NoSQL — mitigado con read replicas (post-MVP)

---

### ADR-2: Drizzle ORM (no Prisma)

**Estado:** activa  
**Fecha:** 2026-05-25

**Contexto:** Type-safety end-to-end. Prisma genera cliente, Drizzle es type-first sobre el schema.

**Decisión:** Drizzle 0.38.

**Consecuencias:**
- ✅ Schema en TypeScript, no en DSL custom
- ✅ Queries con composición tipada `db.select().from(x).where(...)`
- ✅ Sin runtime extra de Prisma (más liviano en Lambda/Edge si llegara el caso)
- ✅ Migrations versionadas en git como SQL legible
- ⚠️ Comunidad más pequeña que Prisma — algunos patterns hay que armarlos a mano

---

### ADR-3: NestJS para la API

**Estado:** activa  
**Fecha:** 2026-05-25

**Contexto:** Estructura modular (módulos por dominio), DI nativa, decoradores, ecosistema (`@nestjs/throttler`, `@nestjs/bullmq`).

**Decisión:** NestJS 11 sobre Express 4 (no Fastify por compatibilidad con módulos comunitarios).

**Consecuencias:**
- ✅ Modules + Controllers + Services bien definidos
- ✅ ThrottlerGuard global con configuración compartida
- ✅ Workaround Express 4 mientras Nest soporta nativo
- ⚠️ Overhead mayor que Express plano — aceptable por la estructura
- ⚠️ `nestjs-doctor` corre antes de cada commit que toque `apps/api`

Detalle: [[Flujo Backend (NestJS)]].

---

### ADR-4: Next.js App Router (no Pages Router)

**Estado:** activa  
**Fecha:** 2026-05-25

**Contexto:** Server Components, layouts anidados, fetching co-localizado.

**Decisión:** Next.js 15 App Router + React 19.

**Consecuencias:**
- ✅ Layouts compartidos + streaming SSR
- ✅ Server Components para metadata pública (futuro)
- ⚠️ React 19 todavía nuevo — algunos peers no compatibles (manejamos caso a caso)
- ⚠️ App Router obliga repensar patrones del Pages Router

Detalle: [[Flujo Frontend (Next.js)]].

---

### ADR-5: Cliente directo de APIs externas

**Estado:** activa  
**Fecha:** 2026-05-25

**Contexto:** Tentación de usar Zapier / Make / n8n para integraciones. **Rechazado.**

**Decisión:** clientes oficiales (Stripe SDK, Agora SDK, arctic OAuth, Firebase Admin) directamente desde `apps/api` y `apps/workers`.

**Consecuencias:**
- ✅ Control total sobre retry, idempotencia, errores
- ✅ Sin dependencia de SaaS intermediario (latencia, downtime ajeno)
- ✅ Costo lineal por uso vs. tier pricing
- ⚠️ Más código a mantener — mitigado con tests unitarios cuando aparezcan

---

### ADR-6: Cloudflare R2 para archivos (diferida)

**Estado:** activa, diferida  
**Fecha:** 2026-05-25

**Decisión:** Cuando se active billing, R2 (no S3). API S3-compatible + sin egress fees.

**Consecuencias:**
- ✅ Sin egress charges (Cloudflare CDN frente)
- ✅ S3 SDK compatible — código portable
- ⚠️ Hoy avatares viven como data URL en DB (cap 150KB) — migración planeada cuando billing R2 activo

Detalle de migración: [[Sprint 1 — Profile, Settings y OAuth]] (sección "Avatar uploader").

---

### ADR-7: Monolito modular (no microservicios)

**Estado:** activa  
**Fecha:** 2026-05-25

**Contexto:** GlobeLiv arranca en Perú, ~MVP. Microservicios prematuros = overhead operativo (CI/CD multi-repo, network latency interna, debugging distribuido).

**Decisión:** Monolito modular hasta 10K+ usuarios activos:
- 1 backend (`apps/api`)
- 1 worker process (`apps/workers`)
- 1 frontend (`apps/web`)
- Compartiendo packages workspace

**Consecuencias:**
- ✅ Deploy unificado, migrations atómicas
- ✅ Type-safety end-to-end sin contratos extra
- ⚠️ Llegando a 10K+ usuarios, evaluamos extraer Streaming Core o Workers a servicio separado

---

### ADR-8: tRPC entre Next y Nest

**Estado:** activa  
**Fecha:** 2026-05-25

**Contexto:** Type-safety sin GraphQL ni REST manual con tipos duplicados.

**Decisión:** tRPC 11 montado como middleware Express en `apps/api` bajo `/trpc/*`. El frontend `apps/web` lo consume vía `@trpc/react-query`.

**Consecuencias:**
- ✅ Cambias procedure en backend → autocompletado en frontend al instante
- ✅ Schemas Zod compartidos en `packages/zod-schemas`
- ✅ Batching automático de queries
- ⚠️ REST controllers Nest (OAuth, webhooks, health) viven en paralelo — documentado por excepción explícita

Detalle: [[Flujo Backend (NestJS)]].

---

### ADR-9: Drizzle migrations versionadas

**Estado:** activa  
**Fecha:** 2026-05-25

**Decisión:** Schema modificado en `.ts` → `pnpm db:generate` → SQL committeado en git → `db:migrate` lo aplica.

**Consecuencias:**
- ✅ Historia auditable
- ✅ Rollback posible (escribir `0004_revert.sql` manual)
- ✅ Mismo workflow en dev / prod
- 🚫 **PROHIBIDO** modificar el schema en prod desde Drizzle Studio sin pasar por migration

---

### ADR-10: Design tokens obligatorios

**Estado:** activa  
**Fecha:** 2026-05-25

**Contexto:** Sin tokens, los colores y tipos se duplican y derivan. Cambios de marca cuestan re-tocar 100 archivos.

**Decisión:** Tailwind 4 `@theme` en `packages/ui/src/tokens.css`. **Prohibido `#hex` o `rgb()` crudos en componentes.**

**Consecuencias:**
- ✅ Cambio de marca = 1 archivo
- ✅ Modo oscuro (cuando llegue) = solo overrides de tokens
- ⚠️ Hace falta disciplina — code review enforce

Detalle de paleta actual ("Kinetic v1"): [[Sprint 1 — Profile, Settings y OAuth]].

---

## 🛠 Reglas operativas (no son ADRs pero son no negociables)

Vienen de `Globeliv/CLAUDE.md §11`:

| Regla | Aplicación |
|---|---|
| **§11.1 Escala como criterio único** | Cualquier propuesta evaluada vs "¿aguanta 10x?" |
| **§11.2 HeroUI primero** | Componentes vía `@globeliv/ui`. Custom solo si HeroUI no llega. Ver [[heroui_first]] |
| **§11.3 Framer Motion con propósito** | Animaciones solo cuando agregan valor real |
| **§11.4 Skills al construir** | Consultar [[Sprint 0 — Skills instalados]] antes de inventar patrones |

---

## 🧭 Bitácora de pivots

> Cuando una ADR cambia, registrar **qué** cambió y **por qué**. Sin esto la deriva del proyecto se vuelve invisible.

Pivots registrados al cierre de Sprint 1-2:

| Fecha | Cambio | Razón |
|---|---|---|
| 2026-05-25 | Identidad inicial pivotó de **phone+OTP** a **email+password** | Defer Twilio costs hasta Sprint 6+. Email es 0-costo |
| 2026-05-25 | `apps/landing` + `apps/app` → **solo `apps/web`** | GlobeLiv no es SaaS; la app es pública desde T+0 |
| 2026-05-27 | Railpack → **Dockerfile multi-stage** | Railpack v0.23 crasheó / sin logs |
| 2026-05-27 | Cookies `sameSite=none + partitioned` → **`sameSite=lax`** | Configurar `api.globeliv.com` simplifica todo el modelo |

Sin pivots de ADRs todavía (las 10 decisiones core siguen activas).

---

## 🔗 Referencias

- `Globeliv/CLAUDE.md` §10 — versión autoritativa
- `Globeliv/ARCHITECTURE_PROMPT.md` — contexto original
- `Globeliv/ROADMAP.md` — pendientes + bitácora de pivots
- [[Visión General del Sistema]] — diagrama maestro
