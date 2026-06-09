---
sprint: 7
nombre: Moderación (reportes, escalamiento, filtro de chat, bloqueo, panel admin)
fecha_inicio: 2026-06-08
fecha_fin: 2026-06-08
estado: Parte A (M0-M4) + M7 completos y desplegados · Parte B (M5/M6) pendiente de credenciales
duración: 1 día
---

# Sprint 7 — Moderación 🛡️ (Parte A + panel admin)

> _"GlobeLiv modera 24/7 sin intervención humana, y el admin recibe alertas. Sin moderación, cualquier proveedor de pagos puede cerrar la cuenta y una sola historia mala destruye la marca — no se lanza al público sin esto."_

**Estado (2026-06-08):** todo el **motor de moderación real** (sin credenciales externas) está construido, verificado y **desplegado a develop**: reportes, escalamiento automático, filtro de chat, bloqueo de suspendidos y **panel admin**. Solo queda la **Parte B**, que depende de credenciales y se construirá en mock como Stripe/FCM: **M5 (detección NSFW)** y **M6 (alertas Telegram/email)**. Commits: `45019b2` (M0+M1), `d7b18d0` (un solo botón de reporte), `abb931c` (M2), `336d46b` (M3+M4), `a108927` (M7).

---

## 🎯 Objetivo (ROADMAP §Sprint 7 + spec Flujo D + Moderación Automática)

Que el sistema modere automáticamente y el admin tenga control. **División acordada con el usuario** en bloques pequeños, separando lo que se puede hacer 100% real ya de lo que necesita credenciales (mock primero).

| # | Entregable | Estado |
|---|---|---|
| **M0** | Schema de moderación (`reports`, `moderation_actions`, `suspended_until`) | ✅ |
| **M1** | Botón ⚑ Reportar en el player (7 razones) | ✅ |
| **M2** | Motor de escalamiento por reportes (3/5/8 + escalamiento de cuenta) | ✅ |
| **M3** | Filtro de chat (palabras prohibidas + anti-spam) | ✅ |
| **M4** | Bloqueo de suspendidos/baneados al transmitir | ✅ |
| **M7** | Panel admin (`/admin`, gateado por rol) | ✅ |
| **M5** | Detección NSFW automática | ⏳ necesita Agora + R2 (mock) |
| **M6** | Alertas al admin (Telegram + email) | ⏳ necesita bot/proveedor (mock) |

---

## 🧱 Decisiones de arquitectura

1. **Cimientos reutilizados:** `user_status` (active/suspended/banned, S0), `chat_messages.isFiltered` (S3, sin lógica hasta hoy), sistema de notificaciones (S6) y `streams.end`/barrido de zombies (mecanismo de corte).
2. **El conteo de cortes para el escalamiento se DERIVA** contando filas `type='cut'` en `moderation_actions` — sin contador denormalizado que se desincronice.
3. **El corte por moderación usa `status='error'`** (no `ended`) para distinguir un fin forzado de uno normal y excluirlo de replays/analytics.
4. **Notificaciones de moderación bypassean el filtro de preferencias** (no son silenciables): se insertan directo + se emiten por socket. Nuevo `notification_type='moderation'` (migración 0014, prioridad `high`).
5. **Reglas configurables sin redeploy vía env** (`MODERATION_*`, `CHAT_*`) con defaults del spec — cambiar en Railway + reiniciar, sin rebuild. (Mover a tabla de config = mejora futura.)
6. **Panel admin embebido en la app web (`/admin`) gateado por rol, NO una app separada.** Montar `admin.globeliv.com` exige crear un proyecto Vercel + DNS del subdominio (acción del usuario); para no bloquearnos, se hizo `/admin` blindado por `adminProcedure` (el gate real está en el backend, que no responde a no-admins). Separarlo es un follow-up de empaquetado.

---

## 🛠 Lo que se construyó (por bloque)

### M0 — Schema (migración `0013`)
- Tabla **`reports`** (streamId/reporterId FK cascade, `reason` enum; índice por stream+fecha para el conteo en ventana; **único parcial (stream, reporter)** = un reporte por usuario).
- Tabla **`moderation_actions`** (histórico inmutable: targetUser, stream, `type` warning/cut/suspension/ban, `source` user_reports/nsfw/chat/manual, reason, suspensionDays, metadata jsonb).
- Columna **`users.suspended_until`**. Enums `report_reason`, `moderation_action_type`, `moderation_source`.

### M1 — Reportar desde el player
- `reportReason` + `createReportInput` (zod) y router `reports.create` (valida stream en vivo, prohíbe auto-reporte, idempotente con índice único).
- `ReportModal` con las **7 razones** del spec (desnudos, violencia, acoso, spam, ilegal, menor, otro).

### M2 — Motor de escalamiento (`moderation-store.ts` → `escalateOnReport`)
- Se dispara tras un reporte **nuevo**. Cuenta reportes en ventana (5 min, configurable). **3 → advertencia** (1 vez/stream), **5 → corte** (UPDATE atómico `live→error`, evita doble corte), **8+ → corte + suspensión ≥7d**.
- **Escalamiento de cuenta** por cortes acumulados: 2º→7d, 3º→30d, 4º+→**ban**.
- Al cortar: registra la acción, **reabre la misión vinculada**, emite `stream:ended reason="moderation"` (saca viewers + quita la card del Home) y **avisa al streamer** (notificación tipo moderation + campana con icono ShieldAlert).

### M3 — Filtro de chat (`chat-moderation.ts`)
- **Palabras prohibidas** (lista base + `CHAT_BANNED_WORDS` por env, normaliza acentos/mayúsculas, match por palabra): el mensaje **no se difunde**, se persiste con `isFiltered=true` (auditoría para el panel) y se avisa al autor. El historial **excluye** los filtrados.
- **Anti-spam por usuario** en Redis: >15 msgs/min y mismo mensaje 3 veces seguidas → bloqueado.

### M4 — Bloqueo de transmisión (`assertCanStream`)
- `streams.create` rechaza a usuarios `banned`/`suspended`. La suspensión se **auto-levanta** de forma perezosa al vencer `suspended_until`.

### M7 — Panel admin (`/admin`)
- `adminProcedure`: verifica `role='admin'` contra la DB **en cada request** (gate real, efecto inmediato sin reemitir token).
- Router `admin`: lecturas (reportes recientes, acciones de moderación, usuarios sancionados) y **acciones manuales** (cortar stream, suspender N días, banear —corta su live—, reactivar). No permite sancionar a otro admin.
- UI `apps/web/src/app/admin`: gateada por `auth.me.role`; pestañas **Reportes / Acciones / Sancionados**; cortar/suspender/banear desde un reporte y reactivar desde sancionados.

---

## 🧩 Fixes del día (8 jun, antes/durante S7)

- **Se quitó el botón "X"/atrás del host** que se percibía como cortar la transmisión; queda solo "Terminar transmisión" (con confirmación). `da61f76`.
- **Modal "Pedir ver un lugar" (Misiones):** se eliminó el campo "título" que confundía con "Lugar" (ahora el título sale del nombre del lugar) y se reemplazó el error de Zod en crudo por un mensaje legible. `3cc28d2`.
- **Notificaciones:** el badge no se limpiaba al abrir el panel; ahora al abrirlo se marcan todas como leídas y solo un evento nuevo vuelve a avisar. `23ba01c`.
- **Un solo botón de reportar:** había dos (el ⚑ sobre el video y uno en la barra de acciones); se dejó solo el ⚑ sobre el video. `d7b18d0`.

---

## 🔬 Verificación

- `typecheck` (10/10) + `lint` (10/10) + `build` (6/6) verdes en **cada bloque** antes de desplegar.
- Migraciones `0013` (tablas) y `0014` (enum `moderation`) aplicadas a Neon local; develop **auto-migra** al arrancar (`RUN_MIGRATIONS_ON_BOOT`).
- _Pendiente de prueba E2E del usuario en `dev.globeliv.com`_ (reportar → advertencia/corte → ver en el panel). Para usar `/admin` hay que promover una cuenta a `role='admin'` en la DB de develop.

---

## 🚀 Deploy (✅ a develop — 2026-06-08)

- Dual-push por cada bloque a `danii-bot/globeliv-app` (Railway/API) + `AdminGlobeliv/Globeliv` (Vercel/web). Último commit `a108927`.
- Railway API + workers desplegaron con éxito; auto-migración aplicó `0013`+`0014`.

---

## 🧾 Deuda técnica / pendientes

- **Parte B necesita credenciales** (se construye en mock cuando se decida):
  - **M5 NSFW:** NSFW.js corre local (gratis), pero capturar frames necesita **Agora Cloud Recording** (Customer ID/Secret + App Certificate) + **bucket R2** (las mismas creds de los replays). Costo recurrente **por minuto de stream**.
  - **M6 alertas:** **Telegram** (bot token + chat id, gratis) y **email** a `yegaf1@gmail.com` (Resend o SMTP).
- **Promover cuenta a admin:** ningún usuario tiene `role='admin'` en develop todavía → `UPDATE users SET role='admin' WHERE lower(email)=lower('…')`.
- **Sin link a `/admin` en la nav** (se entra por URL); agregar un link solo-admin = mejora.
- **Panel admin como app separada** (`admin.globeliv.com`) = follow-up de empaquetado (necesita proyecto Vercel + DNS).
- **Reglas configurables por env** (requiere reinicio) — pasar a tabla de config para "sin redeploy" total.

---

## 🔗 Conexiones

- Viene de: [[Sprint 6 - Retención y Pasaporte (3-5 jun)]]
- Estado en memoria: [[globeliv_s7_moderation_state]]
- Producto / por qué: [[Moderación Automática]], [[Flujo D - Reporte y Moderación]]
- Próximo: cerrar **Parte B (M5 NSFW + M6 alertas)** con credenciales, luego **Sprint 8 — Pulido y Lanzamiento**.
