---
sprint: 2
nombre: Streaming Core
fecha_inicio: 2026-05-27
fecha_fin: en progreso
estado: en-progreso
duración: 2 días (a hoy)
---

# Sprint 2 — Streaming Core 🚧

> _"Que un usuario pueda darle Go Live, otro lo vea, y todo el lifecycle del stream funcione end-to-end con Agora."_

**Estado:** en progreso. Backend y schema completos; frontend Go Live + Watch armado; falta integración real con Agora (todavía corriendo con mock token + mock client) y push a producción.

---

## 🎯 Objetivo

Implementar el flow completo de stream:
1. Usuario logueado abre `/go-live`, pone título, abre cámara, **Iniciar transmisión**
2. Otro usuario abre el Home y ve el stream → click → `/watch/[id]` y se conecta como viewer
3. El streamer puede terminar la transmisión; el viewer ve el "Ya no está en vivo"

Con la base lista, Sprint 3 puede añadir chat + reacciones + propinas sobre este andamiaje.

## ✅ Checklist (en progreso)

- [x] Schema `streams` + enums (`stream_status`, `monetization_level`)
- [x] Migración `0003_productive_ken_ellis.sql` aplicada
- [x] tRPC `streams.create`, `streams.end`, `streams.listLive`, `streams.byId`, `streams.joinAsViewer`
- [x] `TokenService` interface + `MockAgoraTokenService` + `RealAgoraTokenService`
- [x] Singleton `getTokenServiceInstance()` con swap via `AGORA_USE_MOCK`
- [x] Frontend: `/go-live` con preview de cámara local + form (título, monetización, location reverse geocode)
- [x] Frontend: `/watch/[id]` con modo `host` y `viewer`
- [x] Frontend: cliente Agora mock + real (`real-client.ts`, `mock-client.ts`) bajo interfaz común
- [x] Frontend: Home Explorar con `streams.listLive` + filters + StreamCard
- [ ] **Pendiente:** test unitario del `RealAgoraTokenService` con vectores conocidos
- [ ] **Pendiente:** push a `main` y validar con AGORA_USE_MOCK=false en Railway
- [ ] **Pendiente:** capture de thumbnail vía Agora REST (placeholder hasta Sprint 6)
- [ ] **Pendiente:** viewer count realtime (placeholder estático hasta Sprint 3 con Socket.io)

---

## 📦 Lo que se entregó (sub-notas)

- [[Sprint 2 — Schema de Streams]] — tabla `streams`, enums, indexes incluido el partial unique "una live por user"
- [[Sprint 2 — tRPC Streams Router]] — create/end/listLive/byId/joinAsViewer
- [[Sprint 2 — Agora Token Service (mock + real)]] — interface + real + mock + singleton
- [[Sprint 2 — Pantallas Go Live y Watch]] — `/go-live`, `/watch/[id]`, cliente Agora dual

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

> Los commits del Sprint 2 todavía **no han sido pusheados** a `main` al cerrar la jornada del 28-may (modificaciones locales). El historial relevante en git remoto sigue terminando en `cbf268a` (cierre de Sprint 1).

Trabajo del Sprint 2 visible en working tree:

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

## 🚧 Trabajo pendiente para cerrar Sprint 2

- [ ] **Test unitario `RealAgoraTokenService`** con vectores conocidos de Agora (S2-03.8 en plan original)
- [ ] **Set `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE`** en Railway, switch `AGORA_USE_MOCK=false`
- [ ] **Validar end-to-end** con dos browsers / dos tabs: host publica, viewer ve
- [ ] **Manejo de desconexiones** en `useStreamClient` (token expiry, network drop)
- [ ] **Stream end automático** cuando el host cierra el tab (`beforeunload` → `streams.end` keepalive)
- [ ] **`Pantalla 1 - Onboarding`** real (splash con stream live) — el `/welcome` actual es placeholder
- [ ] **Pulido visual** de la Home Explorar — filtros funcionando, empty state, loading state
- [ ] **Push a `main`** que dispare CI verde + deploy Vercel + Railway

---

## 🔗 Pantallas que cubre

- [[Pantalla 1 - Onboarding]] — Home (post-onboarding) muestra streams live; splash final pendiente
- [[Pantalla 2 - Home Explorar]] — implementada (filtros + StreamCard + listLive)
- [[Pantalla 3 - Player de Stream]] — implementada como `/watch/[id]?role=viewer` con chat/tip placeholders
- [[Pantalla 4 - Go Live]] — implementada con preview cámara + form + reverse geocode

## 📋 Estado del producto en progreso

Con `AGORA_USE_MOCK=true` (estado actual local):

1. Login → `/go-live` → abre cámara, pone título, click "Iniciar transmisión"
2. Redirect a `/watch/[streamId]?role=host` — muestra "Estás transmitiendo" con preview
3. En otro browser, `/` muestra el stream → click → `/watch/[streamId]` (modo viewer)
4. Player muestra placeholder "transmisión en vivo (mock)" — no hay video real porque el mock client no se conecta a Agora
5. Host click "Terminar" → `streams.end` → viewer ve "Ya no está en vivo"

Con credenciales reales y `AGORA_USE_MOCK=false`, los pasos 4 muestran video real bidireccional.
