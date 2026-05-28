---
tipo: índice
proyecto: GlobeLiv
inicio: 2026-05-25
actualizado: 2026-05-28
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
| **27-28 may 2026** | Sprint 2 — Streaming Core | 🚧 En progreso | [[Sprint 2 - Streaming Core (27-28 may)]] |

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
└── Sprint 2 - Streaming Core (27-28 may).md
    ├── Sprint 2 — Schema de Streams.md
    ├── Sprint 2 — tRPC Streams Router.md
    ├── Sprint 2 — Agora Token Service (mock + real).md
    └── Sprint 2 — Pantallas Go Live y Watch.md
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
