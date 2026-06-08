---
sprint: 6
nombre: Retención y Pasaporte (push, pasaporte, onboarding, feed, eventos, webcams, replays)
fecha_inicio: 2026-06-03
fecha_fin: 2026-06-05
estado: cerrado y desplegado a develop
duración: 1 día de build de los 8 bloques (3 jun) + deploy con auto-migración y workers + pulido de streaming/nav (4-5 jun)
---

# Sprint 6 — Retención y Pasaporte ✅

> _"Tener un producto que funcione no sirve si los usuarios no vuelven. Este sprint hace que vuelvan: notificaciones, pasaporte con sellos, onboarding sin registro, feed que nunca está vacío, eventos y webcams de respaldo."_

**Estado:** **8/8 bloques completos y desplegados a develop el 2026-06-04** (verificado: "Migraciones aplicadas desde /app/migrations", worker cron sin errores, healthcheck 200). Lo de verdad funciona (notificaciones in-app, pasaporte, eventos, feed priorizado, onboarding); **FCM push, Agora Cloud Recording y Windy webcams corren en MOCK** hasta tener credenciales reales. El **4-5 de junio** se hizo pulido importante de streaming y navegación (ver §pulido). Commits núcleo: `d4dc80b`, `df0bac1`, `0c50be5`, `cabec02`, `bca5d72`, `e3dd652` (3 jun) + `550b561` (auto-migración) + Dockerfile/deploy de workers.

---

## 🎯 Objetivo (ROADMAP §Sprint 6)

**Definición de terminado:** un usuario nuevo tiene onboarding completo, recibe notificaciones relevantes, ve su progreso en el pasaporte, y **la app nunca está vacía** (siempre hay contenido).

| Bloque | Entregable | Estado |
|---|---|---|
| 0 | **Geolocalización ciudad/país** (cimiento del pasaporte) | ✅ |
| 1 | **Notificaciones push (FCM)** | ✅ código · MOCK |
| 2 | **Pasaporte del Explorador** (sellos, niveles) | ✅ |
| 3 | **Replays** automáticos tras cada stream | ✅ código · MOCK |
| 4 | **Onboarding** optimizado (stream antes de registro) | ✅ |
| 5 | **Feed personalizado** con prioridades | ✅ |
| 6 | **Eventos programados** + infra cron | ✅ |
| 7 | **Webcams de respaldo** (Windy) | ✅ código · MOCK |

---

## 🛠 Lo que se construyó (por bloque)

### B0 — Geolocalización ciudad/país
`reverseGeocode()` (Nominatim reverse, cache Redis ~1km). Columnas `city` + `country_code` en `streams` (migración **`0008`**); `streams.create` las puebla. Es el cimiento del Pasaporte.

### B1 — Notificaciones push (FCM) · MOCK
- Tablas `device_tokens` / `notifications` / `notification_preferences` + `users.timezone/locale` (migración **`0009`**). Driver mock/real FCM (`firebase-admin`).
- Worker `send-notification` con **reglas del spec**: máx 2 push/día, ventana 8AM–10PM hora local (Intl + timezone), auto-reducción tras 3 ignoradas, prioridad por tipo. Productor de cola en la API.
- **Puente realtime cross-proceso:** el worker hace `PUBLISH` a un canal Redis → la API lo re-emite a la room `user:{id}` como evento socket `notification:new`.
- **Disparadores cableados:** go-live → seguidores; misión aceptada → creador; propina → streamer; solicitud nueva → streamers cercanos (bounding box de su última transmisión).
- Web: campana + panel de notificaciones, hook listener, toggles en `/settings` persistidos. Push web completo pero dormido sin creds (guardado por `NEXT_PUBLIC_FIREBASE_*`).

### B2 — Pasaporte del Explorador
- Tabla `passport_stamps` (sello único por usuario/país/`lower(city)`, migración **`0010`**). **Niveles derivados** con `COUNT DISTINCT` (sin tabla de niveles): `COUNTRY_LEVELS` (Turista/Conocedor/Experto Local/Embajador) + `GLOBAL_LEVELS` (Curioso/Viajero/Explorador/Ciudadano del Mundo/Leyenda).
- Router `passport`: `recordWatch` (valida ciudad/país, `ON CONFLICT DO NOTHING`, detecta cruce de nivel y encola notif `level_up`), `mine`, `forUser` (público, listo para `/u/[username]`).
- Web: hook `usePassportWatch` dispara `recordWatch` tras **120s** de visualización en foreground (1 request por user/stream máx); `ProfilePassportCard` con banderas + nivel por país.

### B3 — Replays · MOCK
- Columnas en `streams` (migración **`0011`**): `recordingResourceId`, `recordingSid`, `replayUrl`, `replayExpiresAt` + índice parcial. `RecordingService` mock/real (real = Agora Cloud Recording REST con storage R2) inyectado en el context.
- Wiring: `streams.create` dispara `startRecording` fire-and-forget; `streams.end` dispara `stopRecording` (publica `replayUrl` + expiración). `streams.listReplays` (terminados con replay vigente) para fallback de onboarding/feed.
- UI: `StreamEnded` muestra botón "Ver replay" + player inline. _(Replays reales de Agora son HLS .m3u8 → Safari OK; Chrome/FF necesitará `hls.js` como follow-up.)_

### B4 — Onboarding (stream antes de registro)
- `streams.featured`: elige live con más viewers si >5, si no el replay con mayor pico de 24h, si no nada. `WelcomeView` reescrita a hero pantalla completa (reproduce el replay real en loop muted de fondo / gradiente para live), overlay "Esto está pasando ahora en {ciudad}", CTA "Ver stream/replay", botón "Explorar más" a los 5s, prompt de registro sutil a los 30s. Solo a no-autenticados en primera visita.

### B5 — Feed personalizado con prioridades
- `streams.followingLive` (streams live de seguidos). `HomeView` priorizada: 1) Siguiendo en vivo, 2) En vivo/populares (excluye seguidos), y cuando el feed es corto (<8 sin búsqueda): 3) Webcams cercanas, 4) Replays destacados. `StreamCard` ahora acepta `variant` live|replay.

### B6 — Eventos programados + infra cron
- Tabla `scheduled_events` (migración **`0012`**) + router `scheduledEvents` (create/listUpcoming/mine/cancel).
- **Infra cron nueva:** cola BullMQ "cron" con repeatable job cada 60s. `cron.worker.ts` procesa `scheduled-events-reminder`: eventos en los próximos 15min sin recordatorio enviado, reclamados atómicamente, notifica al streamer + seguidores con **doble hora** (tz del usuario y tz del evento).
- UI: página `/agenda` (Próximos / Mis eventos) + `CreateEventModal`. "Agenda" habilitado en navs.

### B7 — Webcams de respaldo (Windy) · MOCK
- Sin tabla DB (desviación justificada: data externa efímera → cache Redis en vez de mirror). `WebcamService` mock/real (real = Windy Webcams API v3, cache Redis 30min). Router `webcams.listNear` (público). `WebcamCard` con badge "Webcam" + crédito obligatorio "Powered by Windy.com".

### Cron Día 1 + resumen semanal (`e3dd652`) — cierra B1 al spec
El cron corre 3 ticks: `scheduled-events-reminder` (60s), `day-one-onboarding` (ventana 4–6h post-registro, claim Redis, copy contextual) y `weekly-digest` (guard 1/semana ISO en Redis).

---

## 🧩 Pulido post-cierre — streaming y navegación (2026-06-04/05)

Tras desplegar los 8 bloques, una ronda fuerte de fixes sobre la experiencia real de transmitir:

- **Preview en vivo en el feed + lista de espectadores + espejo y voltear cámara (`aa61831`, 5 jun):** las cards del Home autoplay el video real del host (muteado) al entrar al viewport, acotado a `MAX_CONCURRENT=6` por escala. Viewer y card **espejan** la cámara frontal del host; botón para voltear cámara frontal↔trasera. Ver [[globeliv_feed_live_preview_decision]] y [[globeliv_camera_mirror_and_flip]].
- **Audio remoto en el viewer (`a36e49d`):** el viewer no reproducía el audio del host.
- **Notificaciones en la nav (`16ed940`→`ff4e36c`):** campana en el header móvil y en el sidebar desktop, item Agenda habilitado y resaltado activo.
- **Auto-terminar streams zombie (`5be8aba`):** un host que abandona (cierra pestaña) ya no deja el stream colgado en el feed; barrido server-side al caer el socket.
- **Estabilidad de streaming (2-3 jun):** códec H264 en vez de VP8 (iOS Safari no publica con VP8), reconexión conservadora + Wake Lock, fix de colisión de uid viewer↔host en Agora, bitrate adaptativo 540p.
- **Cronómetro de transmisión en vivo para el host (`13e7cc1`)** y fix del 404 de "Explorar streams".
- **Sesión que se caía a media transmisión:** `JWT_ACCESS_TTL` subido a 7 días (era 15min → 401 silenciosos que mataban `streams.end`) y botón Terminar resiliente. Ver [[globeliv_auth_no_refresh_flow]].

---

## 🚀 Deploy a develop (✅ 2026-06-04)

- **Auto-migración (`550b561`):** `RUN_MIGRATIONS_ON_BOOT=true` en el servicio API de Railway → la API corre `runMigrations()` al arrancar desde `/app/migrations` (el Dockerfile las copia). **Futuras migraciones se aplican solas al desplegar** — se acabó el `railway run drizzle-kit migrate` manual.
- **Servicio de workers creado:** `Globeliv-workers` en Railway (Dockerfile.workers) — primer build disparado (`d2f92cd`, `cf86880`). El cron y el worker de notificaciones ya corren en develop.
- Migraciones `0008`–`0012` aplicadas. `typecheck` 10/10 + `lint` 10/10 + `build` 6/6.

### ⚠️ Hallazgos de infra (importante, no repetir la confusión)
- **3 repos GitHub divergidos:** canónico + Vercel = `AdminGlobeliv/Globeliv` (remote `adminglobeliv`); **Railway observa `danii-bot/globeliv-app`**; `danii-bot/Globeliv` quedó atrás. → **dual-push:** `git push https://github.com/danii-bot/globeliv-app.git HEAD:develop` (Railway) + `git push adminglobeliv develop` (canónico). Ideal pendiente: reconectar Railway a `AdminGlobeliv/Globeliv`.
- **develop usa la Postgres de RAILWAY**, no Neon — el `.env` local apunta a Neon y es engañoso. Resuelto con la auto-migración.

---

## 🧾 Deuda técnica / pendientes

- **Modos MOCK esperando credenciales reales:** FCM push (B1, service account + `NEXT_PUBLIC_FIREBASE_*`), Agora Cloud Recording (B3, Customer ID/Secret + R2), Windy webcams (B7, API key). El código está listo; solo falta setear `*_USE_MOCK=false` + creds en Railway/Vercel.
- **Borrado FÍSICO de replays expirados en R2** (la expiración lógica ya la cubren las queries) — tick cron por agregar.
- **`hls.js`** para reproducir replays HLS de Agora en Chrome/FF.
- **Pasaporte en el perfil público `/u/[username]`** (`passport.forUser` ya existe, falta cablear la UI).
- **Selector de timezone real** en `CreateEventModal` (hoy usa la del navegador).
- **Refresh-token flow propio:** sigue pendiente (el access JWT ES la sesión; hoy TTL 7d). Producción Railway aún debe subir el TTL antes de lanzar.

---

## 🔗 Conexiones

- Viene de: [[Sprint 5 - Misiones (2-3 jun)]]
- Estado en memoria: [[globeliv_s6_retention_state]], [[globeliv_feed_live_preview_decision]], [[globeliv_camera_mirror_and_flip]], [[globeliv_auth_no_refresh_flow]]
- Producto / por qué: [[Pasaporte del Explorador]], [[Notificaciones Push]], [[Webcams de Respaldo]], [[Replays]], [[Pantalla 1 - Onboarding]]
- Próximo: **Sprint 7 — Moderación** (`apps/admin`, NSFW.js, reports) — o **retomar S4 pagos** con proveedor para Perú.
