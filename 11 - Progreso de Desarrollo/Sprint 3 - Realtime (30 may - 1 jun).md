---
sprint: 3
nombre: Realtime (chat, reacciones, follows, viewer count, filtros Home)
fecha_inicio: 2026-05-30
fecha_fin: 2026-06-01
estado: cerrado
duración: 2 días (30 may – 1 jun)
---

# Sprint 3 — Realtime ✅

> _"Que los viewers puedan chatear y reaccionar durante un stream en vivo, ver cuántos están mirando en tiempo real, y seguir a quien les guste."_

**Estado:** **cerrado y desplegado a develop el 2026-06-01** (arrancó el 30-may, tras [[Sprint 2 - Streaming Core (27-29 may)]]). Los 5 entregables construidos, probados E2E (local **y** contra `api-dev.globeliv.com`), + una ronda de **pulido post-cierre** (UX móvil, follows en vivo, bugs). Commits: `971dd4b` (núcleo), `571b251`, `7e87e41`, `8f06433`, `92d2809`, `be781b6` (pulido).

---

## 🎯 Objetivo (spec §27 + ROADMAP)

Construir la capa de **tiempo real** sobre los streams del Sprint 2. **Definición de terminado:** "los viewers pueden chatear y reaccionar durante streams en vivo, y ven el conteo de espectadores actualizarse solo".

| # | Entregable | Estado |
|---|---|---|
| 1 | **Viewer count en tiempo real** | ✅ |
| 2 | **Chat en vivo** (Socket.io) — incluido el host | ✅ |
| 3 | **Reacciones** (corazones) efímeras | ✅ |
| 4 | **Seguir usuarios** (MVP: solo usuarios) | ✅ |
| 5 | **Filtros / búsqueda en Home** (Popular + Cerca) | ✅ |

---

## 🧱 Decisiones de arquitectura (principio #1: aguantar 10x el tráfico)

Ver [[scale-first-criterion]].

1. **Socket.io con adaptador Redis pub/sub** (no in-memory) — escala horizontal: un emit en un nodo llega a sockets de otro nodo. Dos conexiones IORedis dedicadas (`maxRetriesPerRequest: null`).
2. **Viewer count vía `fetchSockets()` del adaptador** (no un SET manual de presence, como se planeó originalmente). Cuenta membresía across-nodes y **se auto-limpia si un socket cae o un nodo crashea** → sin miembros zombie. El host se excluye filtrando `role`.
3. **El conteo se EXPONE en Redis** (`stream:viewercount:{id}`, `lib/presence.ts`) para que `streams.listLive` lo lea con 1 `MGET` → el Home muestra viewers reales. _Esto cerró la deuda del `viewerCount` estático de Postgres (siempre 0)._
4. **Chat: envío por tRPC, broadcast por WebSocket.** Decisión clave: el `chat.send` entra por tRPC (reusa auth, validación Zod, rate-limit y error-handling del stack HTTP, y respeta "tRPC entre Next y Nest"). El server difunde por WS. El socket sólo **recibe** chat.
5. **Persistencia de chat fuera del hot-path** — los viewers reciben el mensaje por WS sin esperar a Postgres. La escritura (Redis list de últimos 50 + INSERT a `chat_messages`) ocurre después del broadcast.
6. **Reacciones 100% efímeras** — nunca un INSERT por corazón. Throttle de 10/s por socket. Sólo broadcast.
7. **Follows = Postgres** (baja frecuencia). Índice único **parcial** sobre filas activas (permite re-seguir tras unfollow).
8. **Contrato de eventos tipado** en `packages/zod-schemas/src/realtime.ts` — fuente única de verdad; un rename rompe el `typecheck` en cliente **y** server (no falla silencioso en runtime).

---

## 📡 Contrato WebSocket (spec §15)

- **Rooms:** `user:{userId}` (notificaciones personales: tips S4, moderación S7) y `stream:{streamId}` (chat, reacciones, viewer count).
- **Server → client:** `stream:viewer_count {count}`, `chat:message {…, isStreamer}`, `chat:reaction {userId, type}`, `stream:ended {reason}`.
- **Client → server:** `stream:join`, `stream:leave`, `chat:react` _(el chat se envía por tRPC, no por socket)_.

---

## 🛠 Lo que se construyó (por feature)

### 1. Infra realtime + viewer count
- `apps/api/src/realtime/socket-server.ts` — gateway tipado, adaptador Redis, auth opcional por cookie JWT en el handshake (anónimos pueden ver/contar; chat/reacciones exigen login), rooms, recálculo de viewers en join/leave/`disconnecting`.
- `apps/api/src/main.ts` — `initSocketServer()` montado tras `app.listen()` sobre el http.Server de Nest.
- `apps/api/src/lib/cors.ts` — `makeIsAllowedOrigin` extraído y compartido entre Express y el handshake WS.
- `apps/api/src/lib/presence.ts` — exposición del conteo en Redis.
- Web: `lib/realtime/socket.ts` (singleton, `withCredentials`) + hook `use-stream-room.ts`. Wired en `/watch` (3 contadores → realtime).

### 2. Chat en vivo
- Tabla **`chat_messages`** (migración `0004`): `username` denormalizado, `isFiltered` (default false, listo para el filtro de S7), index `(streamId, createdAt)`.
- `apps/api/src/lib/chat-store.ts` — Redis list (historial) + INSERT Postgres + caches de owner y displayName (sin DB-hit por mensaje).
- `apps/api/src/trpc/routers/chat.router.ts` — `send` (valida stream live, difunde, persiste) + `history` (público, Redis→PG).
- Web: hook `use-stream-chat.ts` + componente `stream-chat.tsx` (reemplazó y borró `chat-placeholder.tsx`). **El host también chatea** (input en ambas variantes; su nombre sale en rojo). Sin login → "Inicia sesión para chatear".

### 3. Reacciones (corazones)
- Handler `chat:react` en el gateway: efímero, exige login, throttle 10/s por socket, broadcast `chat:reaction`.
- Web: hook `use-stream-reactions.ts` + `floating-hearts.tsx` (Framer Motion, respeta `prefers-reduced-motion`). Botón de corazón funcional en `player-actions-bar.tsx`; se quitó el botón "Chat" redundante.

### 4. Seguir usuarios (MVP)
- Tabla **`follows`** (migración `0005`): `followedUserId` **NOT NULL** (solo usuarios; las columnas de ubicación del spec se difieren a S6 — YAGNI). Índice único parcial `(followerId, followedUserId) WHERE deleted_at IS NULL`.
- `apps/api/src/trpc/routers/follows.router.ts` — `follow` / `unfollow` (idempotentes) / `status` (isFollowing + contadores).
- Web: `u/[username]/_components/follow-section.tsx` — contadores + botón Seguir/Siguiendo en el perfil público.

### 5. Filtros / búsqueda Home
- `filters.tsx` — chips **Popular** y **Cerca** activados (categorías atardeceres/nightlife/eventos siguen disabled = S6).
- `home-view.tsx` — Popular ordena por viewers; Cerca pide geolocalización (una vez, con aviso si falla) y ordena por distancia haversine. Búsqueda ya existía.
- `streams.listLive` enriquece `viewerCount` con el conteo en vivo de Redis y `streamListItem` ahora lleva `locationLat/locationLng`.

**Criterio de búsqueda/filtros (importante):**
- **Búsqueda** = subcadena (case-insensitive) sobre **título + ubicación(texto) + nombre del streamer** (no solo ubicación). Client-side sobre los ~24 streams cargados.
- **Filtros**: En vivo (el feed ya es solo live) · Gratis (monetización) · Popular (orden por viewers en vivo) · **Cerca (coordenadas GPS auto-detectadas + haversine, NO el texto)**.
- La **ubicación-texto** es etiqueta editable (puede mentir/estar vacía); las **coordenadas GPS** (auto-detectadas en Go Live) son la fuente de verdad para "Cerca", así que el descubrimiento por cercanía es robusto aunque la etiqueta esté mal. Ver _Deuda_ para los huecos.

---

## 🧩 Pulido post-cierre (UX + bugs, 2026-06-01)

- **Chat móvil**: el viewer pasó a una sola pantalla `h-dvh` (video 44dvh + chat con input FIJO abajo). Antes el botón de enviar quedaba enterrado al fondo de una página larga.
- **Seguir + perfil desde la transmisión**: chip del streamer arriba-izq del video (`streamer-bar.tsx`) con avatar+nombre (link a /u/username) + botón Seguir. (Cubre la deuda "botón de seguir en el Player".)
- **Username auto-generado** en signup y OAuth (`lib/username.ts`) — antes era null y los perfiles públicos (/u/[username]) eran inaccesibles. Backfill `user_<shortid>` a usuarios existentes (local + develop).
- **Contador de seguidores real en /profile** (antes "0" hardcodeado en ProfileStatsCard).
- **Aviso en vivo "X te empezó a seguir"**: evento `follow:new` a la room `user:{id}`; aviso flotante sobre el video (`follow-floaties.tsx`). Sólo en follow nuevo.
- **Lista de seguidores/seguidos**: `follows.followers`/`follows.following` + modal con tabs (`follow-list-modal.tsx`) desde el contador del perfil. (Era spec: GET /users/me/followers.)
- **Rediseño Go Live**: pantalla de una sola vista (`h-dvh`) con botón "INICIAR TRANSMISIÓN" FIJO abajo (antes enterrado bajo scroll); monetización compactada (sólo Gratis disponible, lo pagado en una línea); `/go-live` ahora es ruta inmersiva (bare, sin bottom-nav).

---

## 🔬 Verificación (todo E2E en local, con Postgres + Redis reales)

- **Viewer count:** test 2-clientes → 1→2→1; el host (role=host) no suma. ✅
- **Chat:** signup→crear stream→socket join→`chat.send`→broadcast recibido (isStreamer=true)→persistido en Postgres→cacheado en Redis→`chat.history` lo devuelve. ✅
- **Reacciones:** broadcast a ambos (incl. emisor), anónimo bloqueado, throttle 9/20. ✅
- **Follows:** follow/status/idempotencia/anónimo/unfollow/re-seguir/auto-seguir-bloqueado. ✅ UI del perfil renderizada en preview.
- **Filtros:** 2 streams live con viewers reales → `listLive` devolvió viewerCount 3 y 1 (no el 0 estático) + coords. UI del Home en preview: chips Popular/Cerca habilitados e interactivos. ✅
- `typecheck` + `lint` + `build` verdes en todos los paquetes. Se limpiaron warnings `import/order` pre-existentes del Sprint 2.

---

## 🚀 Deploy (✅ completo en develop — 2026-06-01)

- **Código:** commit `971dd4b`. **Ojo — drift de repos:** Vercel (web) despliega de `danii-bot/globeliv-app`; Railway (API) despliega de `danii-bot/Globeliv` (repo distinto). Hubo que pushear a **ambos** (`git push origin develop` + `git push danii-bot develop`). El web desplegó solo (Vercel READY, 971dd4b); la API de Railway requirió el push al repo correcto + el webhook disparó el build (SUCCESS, 971dd4b).
- **Migración aplicada:** develop usa el **Postgres de Railway** (no la Neon local). El deploy NO corre migraciones, así que se aplicaron `0004` (chat_messages) + `0005` (follows) a mano vía `railway run --service Postgres` + `drizzle-kit migrate` (con un RAILWAY_TOKEN temporal). Verificado: las 4 tablas existen, journal de Drizzle con 6 migraciones.
- **Verificado en `api-dev.globeliv.com`:** `follows.status` y `chat.history` responden OK, handshake de Socket.io responde, health db+redis ok. `dev.globeliv.com` HTTP 200. **Develop quedó 100% funcional con S3.**

---

## ✅ Decisiones que se cerraron en este sprint

- **Transporte realtime:** gateway Socket.io **dentro de NestJS** (montado sobre el http.Server existente). Confirmado.
- **Filtro de palabras del chat:** **diferido a Sprint 7** (moderación). El campo `isFiltered` queda listo sin lógica.
- **Seguir ubicaciones:** **diferido a Sprint 6**. MVP solo usuarios.

---

## 🧾 Deuda técnica / pendientes

- **Búsqueda y "Cerca" son client-side sobre los ~24 streams cargados → NO escala.** Para miles de streams en vivo hay que mover a **server-side**: full-text en Postgres + filtro geoespacial (PostGIS / `earthdistance`) + **paginación con cursor**. **Decidido (usuario, 31-may): se difiere a Sprint 8 (perf/escala)** — el MVP client-side es aceptable mientras los streams concurrentes sean pocos.
- **Huecos de ubicación en "Cerca":** (1) si el streamer **niega el permiso de GPS**, su stream no tiene coords y queda al final del orden por cercanía; (2) la etiqueta de texto puede estar mal/vacía (afecta búsqueda por texto y display). Mitigación futura: geocodificar el texto → coords como fallback. Difierido junto con lo anterior.
- **Sync del viewer count a Postgres:** el conteo vive en Redis (y se expone al Home). No se snapshotea `streams.peakViewerCount`/`viewerCount` a Postgres al cerrar el stream — pendiente (worker, S4+).
- **Refresh tokens:** deuda heredada del Sprint 2 — la sesión sigue siendo un solo JWT de 24h. No se atacó en S3.
- **Columnas de ubicación en `follows`:** se omitieron (se agregarán en S6 con seguir-ubicaciones).
- **Drift de repos de deploy:** Railway despliega de `danii-bot/Globeliv`, Vercel de `danii-bot/globeliv-app`. Mitigado con doble-push automático (origin tiene 2 push URLs). Arreglo de raíz (reconectar Railway a globeliv-app) requiere acción del usuario en el dashboard.
- **Verificación visual:** el preview headless del entorno perdía la sesión/recompilaba; varias pantallas con login se verificaron por API + typecheck/lint/build en vez de captura. La verificación visual real la hace el usuario en `dev.globeliv.com`.

> ✅ **Resueltas en el pulido:** migraciones en develop (aplicadas), botón de seguir en el Player (chip del streamer).

---

## 🔗 Conexiones

- Viene de: [[Sprint 2 - Streaming Core (27-29 may)]] / [[Sprint 2 — Cierre y fixes (29 may)]]
- Estado en memoria: [[globeliv-s3-realtime-state]]
- Pantallas que toca: [[Pantalla 3 - Player de Stream]] (chat + reacciones + viewer count), [[Pantalla 2 - Home Explorar]] (filtros + viewer count)
- Próximo: **Sprint 4 — Monetización** (Stripe Connect + propinas 90/10).
