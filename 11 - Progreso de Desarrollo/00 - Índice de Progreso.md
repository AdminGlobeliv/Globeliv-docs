---
tipo: índice
proyecto: GlobeLiv
inicio: 2026-05-25
actualizado: 2026-06-15
---

# 🛠 GlobeLiv — Progreso de Desarrollo

> Bitácora cronológica de **qué se construyó, cuándo y por qué**. Es complementaria a [[00 - Índice]] (que describe el _producto_) — acá vive el _build log_ técnico.

El código vive en `Globeliv/` (carpeta hermana). El spec narrativo vive en este vault. Esta sección registra la traza entre los dos.

---

## 🗓 Línea de tiempo

| Fecha | Sprint | Estado | Nota maestra |
|---|---|---|---|
| **25 may 2026** | Sprint 0 — Setup | ✅ Cerrado | [[Sprint 0 - Setup (25 may)]] |
| **26 may 2026** | Sprint 1 — Auth y Usuarios (arranque + deploy frontend) | ✅ Cerrado | [[Sprint 1 - Auth y Usuarios (26-27 may)]] |
| **27 may 2026** | Sprint 1 — Auth y Usuarios (deploy backend + fixes prod) | ✅ Cerrado | [[Sprint 1 - Auth y Usuarios (26-27 may)]] |
| **27-28 may 2026** | Sprint 2 — Streaming Core (backend, schema, frontend en mock) | ✅ Cerrado | [[Sprint 2 - Streaming Core (27-29 may)]] |
| **29 may 2026** | Sprint 2 — Cierre: Agora real + deploy git-driven + pulido UI/UX | ✅ Cerrado | [[Sprint 2 — Cierre y fixes (29 may)]] |
| **30 may – 1 jun 2026** | Sprint 3 — Realtime (chat, reacciones, follows, viewer count, filtros Home) | ✅ Cerrado y desplegado a develop | [[Sprint 3 - Realtime (30 may - 1 jun)]] |
| **1–2 jun 2026** | Sprint 4 — Pagos y Wallet (Stripe Connect + propinas 90/10) | ⏸️ Code-complete pero **DIFERIDO** (corre en mock; límite Stripe/Perú) | [[Sprint 4 - Pagos y Wallet (1-2 jun)]] |
| **2–3 jun 2026** | Sprint 5 — Misiones (marketplace geolocalizado, el diferenciador #1) | ✅ Cerrado y desplegado a develop | [[Sprint 5 - Misiones (2-3 jun)]] |
| **3–5 jun 2026** | Sprint 6 — Retención y Pasaporte (push, pasaporte, onboarding, feed, eventos, webcams, replays) | ✅ Cerrado y desplegado a develop (FCM/recording/Windy en mock) | [[Sprint 6 - Retención y Pasaporte (3-5 jun)]] |
| **8 jun 2026** | Sprint 7 — Moderación (reportes, escalamiento, filtro de chat, bloqueo, panel admin) | ✅ Parte A (M0-M4) + M7 desplegados · Parte B (M5/M6) pendiente de creds | [[Sprint 7 - Moderación (8 jun)]] |
| **10-15 jun 2026** | Sprint 8 — Pulido, i18n y Previews del Feed (fricción de Misiones/Go Live, internacionalización ES/EN, previews del feed sin blur) | 🔧 En curso · bloques desplegados a develop | [[Sprint 8 - Pulido, i18n y Previews del Feed (10-15 jun)]] |

> **Dónde estamos hoy (15 jun 2026):** sobre el motor ya desplegado (S5 Misiones, S2 Streaming, S7 Moderación), corre el **Sprint 8 de pulido pre-lanzamiento**: se bajó la fricción de **Misiones/Go Live** (autocompletar desde la misión, cámara trasera por defecto, buscador de lugares, geocoding sesgado, feed muestra todas las activas, mínimo 15→10 min), se internacionalizó **toda la app a ES/EN** (511 claves, selector de idioma) y se libró la **guerra contra la "pantalla borrosa / Cargando…"** en las cards del Home (dual-stream de baja resolución, primer frame desde Go Live, placeholder estilo TikTok, revelar video solo con frame no-negro, auto-actualizador). Todo desplegado a develop. Pendientes de fondo siguen: **Parte B de S7** (M5 NSFW + M6 alertas), **retomar S4 pagos** (proveedor para Perú) y los mocks de S6 esperando creds.

---

## 📁 Estructura de esta sección

```
11 - Progreso de Desarrollo/
├── 00 - Índice de Progreso.md       ← estás aquí
│
├── Sprint 0 - Setup (25 may).md     ← nota maestra del sprint
├── Sprint 0 — Monorepo y Arquitectura.md
├── Sprint 0 — Packages y Stack.md
├── Sprint 0 — Skills instalados.md
├── Sprint 0 — DB y Migraciones.md
│
├── Sprint 1 - Auth y Usuarios (26-27 may).md
├── Sprint 1 — Deploy Vercel.md
├── Sprint 1 — Deploy Railway.md
├── Sprint 1 — Sistema de Auth.md
├── Sprint 1 — Profile, Settings y OAuth.md
├── Sprint 1 — CORS y Cookies cross-domain.md
│
├── Sprint 2 - Streaming Core (27-29 may).md
│   ├── Sprint 2 — Schema de Streams.md
│   ├── Sprint 2 — tRPC Streams Router.md
│   ├── Sprint 2 — Agora Token Service (mock + real).md
│   ├── Sprint 2 — Pantallas Go Live y Watch.md
│   └── Sprint 2 — Cierre y fixes (29 may).md
│
├── Sprint 3 - Realtime (30 may - 1 jun).md   ← ✅ cerrado (1 jun)
│
├── Sprint 4 - Pagos y Wallet (1-2 jun).md    ← ⏸️ code-complete, diferido (mock)
├── Sprint 5 - Misiones (2-3 jun).md          ← ✅ cerrado y desplegado (2 jun)
├── Sprint 6 - Retención y Pasaporte (3-5 jun).md  ← ✅ cerrado y desplegado (4 jun)
├── Sprint 7 - Moderación (8 jun).md          ← ✅ Parte A + panel admin (8 jun); M5/M6 pendientes de creds
└── Sprint 8 - Pulido, i18n y Previews del Feed (10-15 jun).md  ← 🔧 en curso (Misiones/Go Live, i18n ES/EN, previews del feed)
```

---

## 🧭 Cómo leer esta bitácora

- Cada **nota maestra de sprint** resume: objetivos, definición de "terminado", commits clave, decisiones, deuda técnica.
- Cada **sub-nota temática** entra en detalle de un componente (deploy, auth, schema, etc.) con referencias a archivos en `Globeliv/`.
- Las fechas son absolutas (`2026-05-25`) para que sigan siendo legibles meses después.
- Los **commits** se identifican por hash corto. Para detalle:
  ```bash
  cd Globeliv && git show <hash>
  ```

---

## 🔗 Conexiones con el resto del vault

- Producto / por qué: [[Qué es GlobeLiv]], [[El Diferenciador]], [[Filosofía del Producto]]
- Pantallas que se construyen: [[Pantalla 1 - Onboarding]], [[Pantalla 3 - Player de Stream]], [[Pantalla 4 - Go Live]], [[Pantalla 6 - Perfil]], [[Pantalla 7 - Configuración]]
- Reglas que se enforcearon en código: [[Reglas de UX que no se negocian]]
- **Cómo funciona el sistema hoy:** [[00 - Índice de Arquitectura]]

---

## 📌 Decisiones de arquitectura tomadas hasta hoy

Estas viven en `Globeliv/CLAUDE.md` como "decisiones que NO se discuten". Las listo acá para tener un punto único de auditoría:

1. **PostgreSQL 17** como BD principal (no Mongo, no Dynamo).
2. **Drizzle ORM** (no Prisma).
3. **NestJS 11 + Express** para la API (no Fastify standalone).
4. **Next.js 15 App Router** (no Pages Router).
5. **tRPC** entre Next y Nest (no GraphQL, no REST manual).
6. **Monorepo modular** con Turborepo + pnpm workspaces (no microservicios hasta 10K+ usuarios).
7. **Cloudflare R2** para archivos (no S3) — _diferido al activar billing_.
8. **Agora SDK** para RTC — el video nunca pasa por nuestros servers.
9. **Vercel** para `apps/web`, **Railway** para `apps/api` y `apps/workers`.
10. **Design tokens obligatorios** — prohibido `#hex` directo en componentes.

Cualquier desviación se registra en la **Bitácora de pivots** dentro de [[ROADMAP]] (ver `Globeliv/ROADMAP.md`).
