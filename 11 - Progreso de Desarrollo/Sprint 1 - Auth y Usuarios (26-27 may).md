---
sprint: 1
nombre: Auth y Usuarios
fecha_inicio: 2026-05-26
fecha_fin: 2026-05-27
estado: cerrado
duración: 2 días
---

# Sprint 1 — Auth y Usuarios

> _"Que un stakeholder pueda entrar a `globeliv.com`, crear cuenta y ver su perfil — todo en producción."_

---

## 🎯 Objetivo

Stack auth completo + perfil + configuración listos para revisión interna, desplegados en Vercel (frontend) y Railway (backend) bajo `globeliv.com` y `api.globeliv.com`.

## ✅ Definición de "terminado"

- [x] Email + password signup/signin con JWT en cookie httpOnly
- [x] Google OAuth (arctic) con account linking por email
- [x] `/profile` editable + avatar upload (client-resize → data URL en DB)
- [x] `/settings` con 5 grupos: Cuenta, Streaming, Notificaciones, Pagos, Privacidad
- [x] `apps/web` live en Vercel (CI: GitHub Actions con prebuilt)
- [x] `apps/api` live en Railway (Dockerfile multi-stage)
- [x] Postgres y Redis productivos en Railway
- [x] Cookies cross-domain funcionando entre `globeliv.com` ↔ `api.globeliv.com`

---

## 📦 Lo que se entregó (sub-notas)

- [[Sprint 1 — Deploy Vercel]] — workflow CI, prebuilt approach, .vercel/project.json
- [[Sprint 1 — Deploy Railway]] — Railpack → Dockerfile, env vars, custom domain
- [[Sprint 1 — Sistema de Auth]] — JWT, bcrypt, signup/signin/logout/me/updateProfile/changePassword/deleteAccount
- [[Sprint 1 — Profile, Settings y OAuth]] — Google OAuth, avatar uploader, design system "Kinetic v1"
- [[Sprint 1 — CORS y Cookies cross-domain]] — partitioned cookies → same-site lax tras subdominio api

---

## 🔑 Commits clave

### 2026-05-26 — Día 1: arranque + CI Vercel

| Hash | Mensaje |
|---|---|
| `99f88e9` / `bf44aea` / `2703866` | `ci: add Vercel deploy workflow (prebuilt approach, no GitHub App needed)` |
| `5958ebb` / `0da20ad` / `8fc532b` | `ci: trigger first Vercel deploy` |
| `d71853b` / `7d0e2b4` / `3d4eb93` | `ci: retrigger Vercel deploy with verified git author` |
| `a4fadfe` | `ci: trigger fresh deploy after cancelling stale runs` |

> 9 commits "trigger" reflejan la iteración con Vercel — debug del prebuilt deploy desde GH Actions hasta que validó author/email correctamente. Lecciones en [[Sprint 1 — Deploy Vercel]].

### 2026-05-27 — Día 2: feature auth completa + Railway + fixes prod

| Hash | Mensaje | Qué hizo |
|---|---|---|
| `cbf268a` | `feat(sprint-1): auth, profile, settings, OAuth y avatar (Fase A+B)` | **El commit grande del sprint.** Backend auth + frontend profile/settings + design system |
| `d60fc73` | `chore(deploy): config Railway/Railpack para apps/api` | Primer intento de deploy backend |
| `c9dbf29` | `chore(deploy): simplificar railpack.json — compatible con Railpack v0.23` | Quitar features que crashearon parser |
| `549c89e` | `chore(deploy): Dockerfile multi-stage para apps/api en Railway` | **Pivot:** Railpack → Dockerfile explícito |
| `60c04cd` | `feat(cors): permitir WEB_PUBLIC_URL y previews *.vercel.app en producción` | CORS para que stakeholders prueben deploys con hash |
| `9533829` | `fix(cors): permitir cualquier subdominio de globeliv.com` | Regex `[\w-]+\.globeliv\.com` (cubre www, admin, api) |
| `491ad23` | `fix(auth): Partitioned cookies para que el frontend Vercel reciba la sesión` | Chrome 2024+ bloquea third-party cookies |
| `47cac4c` | `refactor(auth): cookies same-site lax ahora que api vive en api.globeliv.com` | Tras configurar subdominio: same-site = lax, quitar partitioned |

---

## 🧠 Decisiones que se tomaron

1. **Email + password como identidad inicial** (no OTP por SMS). Razón: defer Twilio costs hasta Sprint 6+; email es 0-costo. `phone` se reintroduce con 2FA.
2. **JWT en cookie httpOnly** (no en localStorage). Razón: XSS resistance + el frontend Next no necesita leer el token (todas las llamadas son tRPC desde el cliente y el cookie viaja solo).
3. **Avatar como data URL en DB** (cap 150KB tras resize client a 256×256 JPEG). Razón: no hay billing aún para Cloudflare R2; migración planeada cuando se active R2. Riesgo asumido: row weight más alto.
4. **Sin refresh tokens en Sprint 1.** Access token TTL extendido (días). Refresh + Redis allowlist con `jti` entra en Sprint 2+.
5. **`raw body` en `/webhooks` antes de `express.json()`** — preparado para Stripe webhook HMAC en Sprint 4. Montado de día 1 para no olvidarlo.
6. **CORS por regex + WEB_PUBLIC_URL** — cualquier subdominio de `globeliv.com` + previews `*.vercel.app`. Stakeholders prueban PRs sin re-configurar CORS.
7. **Dockerfile multi-stage en lugar de Railpack** — más confiable, build logs visibles, mismo Node 22 alpine. Detalles en [[Sprint 1 — Deploy Railway]].
8. **Workaround NestJS 11 + Express 4** — `ExpressAdapter.isMiddlewareApplied` override. Documentado en [[Sprint 1 — Sistema de Auth]]. Quitar cuando Nest soporte Express 4 nativamente.
9. **Design tokens "Kinetic v1"** — pivot de `--brand-red: #ff3344` a `#ff3b4e` + paleta navy + gradiente + fuentes Inter + Space Grotesk. Más detalle en [[Sprint 1 — Profile, Settings y OAuth]].

---

## 🗄 Migraciones añadidas

- **`0001_flat_zarda.sql`** (26 may, 20:38) — `users.location text`, unique index funcional `lower(username)`
- **`0002_amazing_whistler.sql`** (26 may, 22:18) — `password_hash` nullable (para users solo-OAuth), `google_id text` + unique index

---

## 🚧 Deuda técnica abierta al cerrar Sprint 1

- [ ] **Avatares en data URL** → migrar a Cloudflare R2 cuando haya billing
- [ ] **Refresh tokens** con Redis allowlist (`session:{jti}`) — Sprint 2+
- [ ] **Fix `@source` de HeroUI** para Tailwind 4 + pnpm symlinks (workaround actual: custom `TextInput`, `SettingsItem`, etc.). Está rompiendo el ideal de "HeroUI primero" — ver [[heroui_first]]
- [ ] **`workflow_dispatch` con concurrency lock** — no cancelamos deploys en curso, ¿afecta UX si se quiere retrigger?
- [ ] Tests E2E del flow signup → /profile → /settings (Playwright) — diferido

---

## 🔗 Pantallas que cubre

- [[Pantalla 6 - Perfil]] — implementada
- [[Pantalla 7 - Configuración]] — implementada (UI sin lógica de wallet aún)
- [[Pantalla 1 - Onboarding]] — `/welcome` placeholder, pulido real en Sprint 2

## 📋 Estado del producto al cerrar Sprint 1

Un stakeholder puede ya:

1. Entrar a `https://www.globeliv.com`
2. Crear cuenta con email + password (o con Google OAuth)
3. Ir a `/profile`, editar display name + location + username + avatar
4. Ir a `/settings`, ver los 5 grupos (toggles funcionando, sin persistencia aún de algunas opciones)
5. Hacer logout y volver a entrar

Lo que **NO** funciona todavía (entra en Sprint 2): crear stream, ver stream, chat, propinas.
