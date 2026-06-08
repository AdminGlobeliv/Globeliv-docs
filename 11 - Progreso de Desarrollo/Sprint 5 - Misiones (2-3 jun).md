---
sprint: 5
nombre: Misiones (marketplace geolocalizado de solicitudes)
fecha_inicio: 2026-06-02
fecha_fin: 2026-06-03
estado: cerrado y desplegado a develop
duración: 1 día de build (2 jun) + fixes de ciclo de vida y realtime (3 jun)
---

# Sprint 5 — Misiones ✅ (el diferenciador #1)

> _"Un viewer pide ver un lugar del planeta → los streamers cercanos lo ven en su feed → el primero que transmite gana la misión → al cumplir se marca completada."_

**Estado:** **cerrado y desplegado a develop el 2026-06-02** (commit `f5659f1`, junto al código mock de [[Sprint 4 - Pagos y Wallet (1-2 jun)|S4]]). El **marketplace bidireccional geolocalizado** — el moat del producto, lo que no tienen Twitch/TikTok/IG/YouTube Live — vivo en `dev.globeliv.com`. El **3 de junio** se hizo una ronda de fixes de ciclo de vida y de realtime (commits `524373e`, `e49f3de`, `6a9b610`).

---

## 🎯 Objetivo (ROADMAP §Sprint 5 + spec §Flujo B, Pantalla 5)

**El sprint que activa el moat.** Antes de esto GlobeLiv es "una app de streaming más"; después, es algo único. **Definición de terminado:** un viewer crea una solicitud → un streamer cercano la ve → la acepta → transmite desde el lugar → al completarse se marca cumplida.

| # | Entregable | Estado |
|---|---|---|
| 1 | **CRUD de solicitudes** (crear/ver/aceptar/completar) | ✅ |
| 2 | **Feed cercano** (geolocalizado) | ✅ |
| 3 | **Aceptar y vincular** la solicitud al stream | ✅ |
| 4 | **Aviso a streamers cercanos** (in-app/realtime) | ✅ |
| 5 | **Pantalla "Misiones"** funcional | ✅ |
| — | Escrow de recompensas pagadas | ⏸️ difiere con S4 (proveedor de pagos) |

---

## 🧱 Decisiones de arquitectura (confirmadas con el usuario, 2026-06-02)

1. **Geocoding gratis con Nominatim (OpenStreetMap)** — SIN llave ni cuenta externa (a diferencia de Stripe). El usuario escribe "Plaza de Armas, Cusco" → coords. Cero fricción de setup. Cache Redis 30 días, timeout 4s, best-effort.
2. **Solo solicitudes "solo propinas" ($0) en el MVP.** Las recompensas pagadas ($3/$5/$10 con escrow) dependen del proveedor de pagos (diferido con S4). La columna `reward_amount` y los botones pagados (disabled) quedan listos.
3. **El claim de la misión es atómico AL INICIAR LA TRANSMISIÓN** (`streams.create` con `requestId`), no en un paso "accept" separado → "primero en transmitir gana", sin estado intermedio colgable.
4. **Aviso a streamers cercanos = in-app/realtime** (broadcast por socket). El push FCM es Sprint 6.

---

## 🛠 Lo que se construyó (por fase G0→G7)

### G0 — Schema
- Migración **`0007`**: tabla **`requests`** (creador FK cascade, `title`/`description`/`locationName`, `locationLat/Lng` numeric nullable, `rewardAmount` default 0, `status` enum, `radiusKm` default 5, `acceptedById`, `expiresAt` NOT NULL; índices status/location/user/expires + **UNIQUE PARCIAL one-active-per-streamer**) + enum `request_status` (open/accepted/live/completed/expired/cancelled) + columna **`streams.request_id`** (el vínculo stream↔solicitud vive en streams, por spec).
- zod-schemas `requests.ts`: inputs y `requestListItem` con creador embebido.

### G1 — Crear + geocoding
- `geocode.ts`: `geocodePlace(query)` vía Nominatim con User-Agent (política OSM), cache Redis 30d.
- Router `requests.create`: fuerza `rewardCents=0` (MVP), usa coords del cliente o geocodifica el nombre, `expiresAt = now + 24h`.
- UI: página `/misiones` + `create-request-modal.tsx`. "Misiones" habilitado en las 3 navs.
- **Verificado E2E:** "Plaza de Armas, Cusco" → geocodificó a `-13.5168, -71.9788` (coords reales). ✓

### G2 — Feed
- `requests.list` (público): abiertas + vigentes (expiración perezosa) + filtro geográfico por **bounding box**, newest first.
- **Bug importante evitado:** la columna numeric mapea a `string` en TS — comparar el bounding box con `String(n)` hace que Postgres compare como **texto** y con coords negativas el orden lexicográfico se invierte. Solución: comparar con `sql\`${col} >= ${number}\`` (bindea número). Aplica a cualquier filtro futuro sobre lat/lng.
- UI: feed real con geolocalización del navegador, distancia haversine por card, `request-card.tsx` con botón "Aceptar y transmitir".
- **Verificado E2E:** list cerca de Cusco (50km)=1, cerca de Lima (50km)=0 (bounding box correcto). ✓

### G3 — Aceptar y transmitir
- `streams.create` ahora acepta `requestId` y corre en **transacción**: si hay `requestId`, `UPDATE requests SET status='live', acceptedById WHERE status='open' AND user_id<>yo AND expiresAt>now RETURNING`; si 0 filas → `CONFLICT "Esta misión ya fue tomada"` (rollback, no crea stream). `requests.byId` para el contexto en Go Live.
- Go Live lee `?request`, muestra banner verde "Cumpliendo una misión", pre-llena ubicación con SUS coords e incluye `requestId` al transmitir.
- **Verificado E2E:** A crea → B transmite → request a 'live'; C intenta la misma → CONFLICT; D intenta auto-aceptarse → CONFLICT. ✓

### G4 — Ciclo de vida (completar/expirar/cancelar)
- `resolveRequestOnStreamEnd`: al terminar el stream vinculado, ≥ umbral → `completed`; por debajo → reabre a `open` y libera al accepter.
- `sweepExpiredRequests`: marca open vencidas a `expired`, throttled con lock Redis 1/min (evita write-storm).
- `requests.cancel`: el creador cancela solo si está `open` (atómico); si ya fue aceptada → `PRECONDITION_FAILED`.

### G5 — Mis solicitudes
- `requests.myCreated` + `requests.myAccepted`. UI con HeroUI **Tabs** (Cerca / Mis solicitudes / Acepté), `request-status-badge.tsx` y botón Cancelar.

### G6 — Aviso a streamers cercanos (in-app)
- Evento realtime `request:new`. `requests.create` hace broadcast global por socket (best-effort).
- `use-mission-alerts.ts` escucha, ignora las propias, y solo avisa si la misión está ≤50km de la ubicación cacheada en `localStorage` (haversine) — NO pide permiso de geo app-wide. Toast flotante `mission-alerts.tsx` montado app-wide.
- **Verificado E2E:** el cliente socket recibió el broadcast con coords geocodificadas (Arequipa). ✓

### G7 — Deploy
- Commit `f5659f1` (S4+S5, 55 archivos) en develop. Doble-push a globeliv-app (Vercel) + Globeliv (Railway) → ambos auto-desplegaron. Migraciones `0006` (S4) + `0007` (S5) aplicadas a la Postgres de Railway develop.

---

## 🧩 Fixes de ciclo de vida y realtime (2026-06-03)

- **Realtime de misiones (`524373e`):** el creador no veía su misión hasta entrar a "Mis solicitudes" y los cercanos la veían con retraso. Fix: `requests.create` ahora invalida caché en `onSuccess`; al llegar `request:new` cercano se invalida `requests.list`; el feed usa `staleTime:0` + `refetchOnMount:"always"`.
- **Umbral de cumplimiento 5 → 15 min (`e49f3de`):** una misión se da por cumplida con ≥15 min de transmisión (antes 5). Por debajo se reabre para todos. Banner en go-live: "Transmite al menos 15 min para darla por cumplida". Aviso al streamer al cumplirse.
- **No cancelar una misión ya aceptada (`6a9b610`):** la UI oculta el botón Cancelar cuando la misión dejó de estar `open`.

---

## 🔬 Verificación

- E2E por fase (G1–G6) contra Postgres + Redis reales, datos de prueba limpiados tras cada uno.
- `typecheck` 10/10 + `lint` verdes + `next build` web 11/11 páginas.
- En develop: `requests.list` responde (tablas existen), `dev.globeliv.com/misiones` 200, Railway API deploy SUCCESS.

---

## 🧾 Deuda técnica / pendientes

- **`request:new` es broadcast GLOBAL** a todos los sockets (a 10k+ es derrochador) → migrar a **push geo-dirigido** (FCM, S6) o geo-rooms. Por ahora el payload es chico y el cliente filtra.
- **Geocoding Nominatim corre SÍNCRONO** en `requests.create` (hasta 4s, cacheado). Mover a job/optimistic a futuro.
- **Recompensas pagadas + escrow:** difieren con el proveedor de pagos (S4).
- **`sweepExpiredRequests` es fire-and-forget con lock** — a escala migrar a job BullMQ.
- **Revocar el `RAILWAY_TOKEN` temporal** del deploy.

---

## 🔗 Conexiones

- Viene de: [[Sprint 4 - Pagos y Wallet (1-2 jun)]]
- Estado en memoria: [[globeliv-s5-requests-state]]
- Producto / por qué: [[El Diferenciador]], [[Sistema de Misiones]], [[Flujo B - Crear una Solicitud]]
- Pantalla: [[Pantalla 5 - Misiones]]
- Próximo: [[Sprint 6 - Retención y Pasaporte (3-5 jun)]].
