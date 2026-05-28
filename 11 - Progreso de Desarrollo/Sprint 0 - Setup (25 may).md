---
sprint: 0
nombre: Setup
fecha_inicio: 2026-05-25
fecha_fin: 2026-05-25
estado: cerrado
duración: 1 día
---

# Sprint 0 — Setup

> _"Antes de construir cualquier feature, dejar listo el esqueleto: monorepo, packages, CI y un Hello World desplegable."_

---

## 🎯 Objetivo

Tener el **scaffold del monorepo en producción**, `pnpm typecheck` verde y `apps/web` live en Vercel.

## ✅ Definición de "terminado"

- [x] Monorepo con pnpm workspaces + Turborepo
- [x] `packages/{config-typescript, config-eslint, database, zod-schemas, ui}` con stubs reales
- [x] `apps/api` (NestJS 11 + Express + tRPC + helmet + throttler + health check)
- [x] `apps/workers` (BullMQ + IORedis + Pino con PII redaction)
- [x] `apps/web` (Next.js 15 + HeroUI + Tailwind 4 + design tokens)
- [x] CI en GitHub Actions: lint + typecheck + build
- [x] `CLAUDE.md` + `ROADMAP.md`
- [x] Migration inicial de `users` (Drizzle)

---

## 📦 Lo que se entregó (sub-notas)

- [[Sprint 0 — Monorepo y Arquitectura]] — estructura `apps/`, `packages/`, Turborepo, pnpm workspaces
- [[Sprint 0 — Packages y Stack]] — dependencias y versiones exactas instaladas
- [[Sprint 0 — Skills instalados]] — agentes Claude (UI/UX, backend, NestJS doctor)
- [[Sprint 0 — DB y Migraciones]] — schema inicial `users` + Drizzle migration 0000

---

## 🔑 Commits clave (2026-05-25)

| Hash | Mensaje | Qué hizo |
|---|---|---|
| `a0ee8d1` | `chore: scaffold Sprint 0 monorepo per ARCHITECTURE_PROMPT` | Commit fundacional del repo |
| `c793e61` | `fix(database): drop .js extensions on internal schema imports` | Arregla imports de schema + agrega migration `0000_complete_khan.sql` (luego renombrada) |
| `1b7c182` | `refactor: rename apps/landing → apps/web (drop SaaS-style landing split)` | Decisión: GlobeLiv no es SaaS con landing separada; la app es pública desde T+0 |
| `eb706bf` | `chore: install Claude skills + add scale/HeroUI/Framer rules to CLAUDE.md` | 17 skills backend + reglas operativas |
| `0080d65` | `docs: install UI UX Pro Max skills (manual, /plugin not available)` | Skills UI/UX a nivel proyecto |

---

## 🧠 Decisiones que se tomaron

1. **Una sola app frontend (`apps/web`).** Se descartó la separación `apps/landing` + `apps/app` típica de SaaS — la pantalla 1 (splash con stream live) **es** la home pública. Razón: GlobeLiv es app-first, no marketing-first. Ver [[Pantalla 1 - Onboarding]].
2. **Node 22 LTS** (24.x dev OK) y **pnpm 11.3.0** — fijo en `engines` y en `packageManager` del `package.json` raíz para evitar drift entre máquinas.
3. **PII redaction** en logs desde el día 1 — Pino con `redact` config para `email`, `phone`, `password`, `token`, `authorization`.
4. **Tipos compartidos vía workspace, no vía build artifact.** `@globeliv/zod-schemas` y `@globeliv/database` se importan como source (no `dist/`) → cambios se reflejan en typecheck inmediato.
5. **`tsup` para `apps/api` y `apps/workers`** (output ESM), **Next compiler** para `apps/web`. Mantenemos ESM en runtime de backend para alinearse con Node 22.

---

## 🚧 Deuda técnica abierta al cerrar Sprint 0

- [ ] **Plugin ESLint custom** que falle si una query no incluye `user_id` (multi-tenant guard). Diferido — se prioriza cuando aparezca la primera tabla `user_id`-scoped (Sprint 2 con `streams` lo enforce manual).
- [ ] **PII redaction reforzada** — hook que detecte teléfonos en `text/rawInput`. Diferido a Sprint 6 con FCM.
- [ ] Crear rol+BD Postgres local (SQL manual) — _completado al cerrar Sprint 1 con Postgres en Railway_.

---

## 🔗 Referencias

- Documento maestro de arquitectura: `Globeliv/CLAUDE.md`
- Roadmap completo: `Globeliv/ROADMAP.md`
- Spec original: `GLOBELIV_Especificacion.md`
- ADRs: `ARCHITECTURE_PROMPT.md`
