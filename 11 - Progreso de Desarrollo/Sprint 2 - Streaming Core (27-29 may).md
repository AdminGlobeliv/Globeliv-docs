---
sprint: 2
nombre: Streaming Core
fecha_inicio: 2026-05-27
fecha_fin: 2026-05-29
estado: cerrado
duración: 3 días
---

# Sprint 2 — Streaming Core ✅

> _"Que un usuario pueda darle Go Live, otro lo vea, y todo el lifecycle del stream funcione end-to-end con Agora."_

**Estado:** ✅ **CERRADO el 2026-05-29.** Backend, schema, frontend Go Live + Watch y **Agora REAL** funcionando end-to-end en develop (`dev.globeliv.com` + `api-dev.globeliv.com`). La jornada del 29-may resolvió la integración Agora real, el deploy git-driven y una tanda grande de pulido UI/UX. Detalle en [[Sprint 2 — Cierre y fixes (29 may)]]. Sprint 3 ([[Sprint 3 - Realtime (30 may - 1 jun)]]) construye chat + reacciones + viewer count realtime sobre esta base.

---

## 🎯 Objetivo

Implementar el flow completo de stream:
1. Usuario logueado abre `/go-live`, pone título, abre cámara, **Iniciar transmisión**
2. Otro usuario abre el Home y ve el stream → click → `/watch/[id]` y se conecta como viewer
3. El streamer puede terminar la transmisión; el viewer ve el "Ya no está en vivo"

Con la base lista, Sprint 3 puede añadir chat + reacciones + propinas sobre este andamiaje.

## ✅ Checklist (cerrado)

- [x] Schema `streams` + enums (`stream_status`, `monetization_level`)
- [x] Migración `0003_productive_ken_ellis.sql` aplicada
- [x] tRPC `streams.create`, `streams.end`, `streams.listLive`, `streams.byId`, `streams.joinAsViewer`
- [x] `TokenService` interface + `MockAgoraTokenService` + `RealAgoraTokenService`
- [x] Singleton `getTokenServiceInstance()` con swap via `AGORA_USE_MOCK`
- [x] Frontend: `/go-live` con preview de cámara local + form (título, monetización, location reverse geocode)
- [x] Frontend: `/watch/[id]` con modo `host` y `viewer`
- [x] Frontend: cliente Agora mock + real (`real-client.ts`, `mock-client.ts`) bajo interfaz común
- [x] Frontend: Home Explorar con `streams.listLive` + filters + StreamCard
- [x] **Agora REAL configurado (29-may):** App ID `52dcf7d5…`, `AGORA_USE_MOCK=false` en Railway develop + Vercel Preview. Token real verificado (prefijo `007…`)
- [x] **Deploy git-driven (29-may):** push a `develop` → auto-deploy web (Vercel) + API (Railway) desde `danii-bot/globeliv-app`
- [x] **Pulido UI/UX (29-may):** tema dark HeroUI, host full-screen tipo TikTok, modal de finalizar, preview casi-en-vivo en el feed, espejo de selfie, fix botón "Iniciar transmisión"
- [ ] **Diferido a Sprint 3:** viewer count realtime (placeholder estático → Socket.io)
- [ ] **Diferido a Sprint 6:** capture de thumbnail vía Agora REST
- [ ] **Deuda:** test unitario del `RealAgoraTokenService` con vectores conocidos; setup Agora en **producción** (env Production de Railway/Vercel NO lo tiene)

---

## 📦 Lo que se entregó (sub-notas)

- [[Sprint 2 — Schema de Streams]] — tabla `streams`, enums, indexes incluido el partial unique "una live por user"
- [[Sprint 2 — tRPC Streams Router]] — create/end/listLive/byId/joinAsViewer
- [[Sprint 2 — Agora Token Service (mock + real)]] — interface + real + mock + singleton
- [[Sprint 2 — Pantallas Go Live y Watch]] — `/go-live`, `/watch/[id]`, cliente Agora dual
- [[Sprint 2 — Cierre y fixes (29 may)]] — Agora real en develop, deploy git-driven y la tanda de pulido UI/UX que cerró el sprint

---

## 🧠 Decisiones que se tomaron

1. **Mock client + Mock token service desde el día 1.** El frontend completo (Player, Go Live, end-to-end) se puede demostrar **sin credenciales Agora** — útil para CI y para iterar UI sin gastar minutos. La interfaz `TokenService` permite swap atómico via env.
2. **"Una transmisión live por user" enforced a nivel DB** — partial unique index `WHERE status='live' AND deleted_at IS NULL`. El router lo intenta y mapea `23505` (PG unique violation) a `CONFLICT`. Mejor que validar en código (race condition entre `SELECT` y `INSERT`).
3. **Monetización pagada bloqueada hasta tener métricas de rating** — `streams.create` lanza `PRECONDITION_FAILED` para cualquier `monetization != "free"`. Mensaje user-facing referencia la regla del spec (≥ 4.5⭐, ≥ 20 streams). UI también pinta candados sobre los botones.
4. **`/watch/[id]` con `joinAsViewer` público.** No exigir login para ver un stream = share-link friendly. Si el viewer está logueado el `uid` es estable; si no, `uid` efímero — aceptable para MVP. Sprint 3 puede endurecerlo si aparece abuso.
5. **`streams.byId` SIN token, `streams.joinAsViewer` SÍ token.** Lectura (cacheable) separada de emisión (efecto colateral) → frontend puede prefetch metadata sin gastar privilegios Agora.
6. **`agora_channel_id` UUID generado server-side.** No usamos el `stream.id` directo para no leakear identificadores internos en el SDK de Agora.
7. **`userIdToUid(uuid) -> uint32`** — Agora exige uid numérico. Hash determinístico estable per-user.
8. **`viewer_count`/`peak_viewer_count` se actualizan vía SQL directo en Sprint 2.** Sprint 3 introduce Socket.io + Redis pub/sub y se moverá a hot-path Redis con snapshot a DB al cerrar.
9. **`hostToken` se persiste en `sessionStorage` por streamId** entre `/go-live` y `/watch/[id]?role=host`. Limpia al cerrar tab → no quedan tokens huérfanos. `sessionStorage` bloqueado (modo privado) → mensaje claro en Player.

---

## 🔑 Commits clave

> **Actualización 29-may:** el trabajo del Sprint 2 ya está **pusheado a `develop`** (repo `danii-bot/globeliv-app`) y desplegado a `dev.globeliv.com` + `api-dev.globeliv.com` vía auto-deploy. El día 28 quedó como modificaciones locales; el 29 se resolvió el deploy git-driven y se subió todo. Commit base del streaming core: `778df66` (`feat(sprint-2): streaming core con Agora + shell global de navegación`). Cierre del sprint el 29-may en `d056d58`.

Trabajo del Sprint 2 (28-may) visible en el primer push:

- `packages/database/migrations/0003_productive_ken_ellis.sql` (27-may 16:30)
- `packages/database/src/schema/streams.ts`, `schema/enums.ts` (27-may 16:30)
- `packages/zod-schemas/src/streams.ts`, `src/jobs/*` (27-may)
- `apps/api/src/trpc/routers/streams.router.ts` (27-may)
- `apps/api/src/modules/streams/` (mock + real + interface + uid + instance + module)
- `apps/web/src/app/go-live/` (27-may)
- `apps/web/src/app/watch/[id]/` (27-may)
- `apps/web/src/lib/agora/` (cliente dual)
- `apps/web/src/app/_components/{stream-card,filters}.tsx` (Home Explorar)
- `apps/web/src/lib/geo/use-reverse-geocode.ts` (reverse-geocode para location en `/go-live`)

> El commit de cierre del Sprint 2 se etiquetará probablemente `feat(sprint-2): streaming core (Go Live + Player + Home)` siguiendo la convención del `cbf268a` de Sprint 1.

---

## 🗄 Migraciones añadidas

- **`0003_productive_ken_ellis.sql`** (2026-05-27, 16:30) — crea enums `stream_status`, `monetization_level` y la tabla `streams` con sus 4 indexes (channel unique, status+started_at, user+started_at, partial unique "one live per user").

---

## 🚧 Resolución de pendientes (al cierre 29-may)

- [x] **Set `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE`** en Railway develop, switch `AGORA_USE_MOCK=false` (+ Vercel Preview `NEXT_PUBLIC_AGORA_*`)
- [x] **Validar end-to-end** con Agora real en develop — token `007…` emitido y verificado
- [x] **Push y deploy** — auto-deploy a `dev.globeliv.com` desde `develop` (resuelto el repo propio `danii-bot/globeliv-app`)
- [x] **Pulido visual** de la Home Explorar y de `/watch` — feed con preview, host full-screen, modal de finalizar

**Deuda técnica que se arrastra (no bloqueante):**

- [ ] **Test unitario `RealAgoraTokenService`** con vectores conocidos de Agora (S2-03.8 en plan original)
- [ ] **Manejo de desconexiones** en `useStreamClient` (token expiry, network drop)
- [ ] **Stream end automático** cuando el host cierra el tab (`beforeunload` → `streams.end` keepalive)
- [ ] **`Pantalla 1 - Onboarding`** real (splash con stream live) — el `/welcome` actual es placeholder
- [ ] **Agora en producción** — los env de Production (Railway/Vercel) NO tienen Agora; repetir el setup si se quiere streaming en prod
- [ ] **Refresh tokens** — hoy la sesión es un solo JWT de 24h (subido desde 15min); falta `auth.refresh` + auto-renovación (heredado a Sprint 3+)

---

## 🔗 Pantallas que cubre

- [[Pantalla 1 - Onboarding]] — Home (post-onboarding) muestra streams live; splash final pendiente
- [[Pantalla 2 - Home Explorar]] — implementada (filtros + StreamCard + listLive)
- [[Pantalla 3 - Player de Stream]] — implementada como `/watch/[id]?role=viewer` con chat/tip placeholders
- [[Pantalla 4 - Go Live]] — implementada con preview cámara + form + reverse geocode

## 📋 Estado del producto al cierre (29-may)

En **develop** (`dev.globeliv.com`) con `AGORA_USE_MOCK=false` — **video real**:

1. Login → `/go-live` → abre cámara, pone título, click "Iniciar transmisión" (con hint de requisitos si el botón está deshabilitado)
2. Redirect a `/watch/[streamId]?role=host` — host full-screen tipo TikTok con su cámara
3. En otro browser, `/` (feed) muestra el stream con preview casi-en-vivo → click → `/watch/[streamId]` (modo viewer ve el **video real** del host)
4. La X del host abre el modal de confirmación → "Terminar" → `streams.end` → viewer ve la pantalla de "fin de transmisión"

El path con `AGORA_USE_MOCK=true` sigue disponible para CI / iterar UI sin gastar minutos de Agora.

> Nota operativa: si un stream queda "pegado" en `live`, se cierra a mano vía `psql` contra `DATABASE_PUBLIC_URL` (Railway Postgres): `UPDATE streams SET status='ended', ended_at=now() WHERE status='live'`.
