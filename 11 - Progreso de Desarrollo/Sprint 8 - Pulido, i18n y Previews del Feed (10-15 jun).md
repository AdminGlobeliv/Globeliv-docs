---
sprint: 8
nombre: Pulido, i18n y Previews del Feed
fecha_inicio: 2026-06-10
fecha_fin: 2026-06-15
estado: En curso — bloques de Misiones, i18n ES/EN y Previews del feed desplegados a develop
duración: 5 días (10, 11, 12, 13 y 15 jun)
---

# Sprint 8 — Pulido, i18n y Previews del Feed ✨

> _"Antes de abrir al público hay que quitar fricción: crear una misión sin pelear con el formulario, transmitir sin pantallas borrosas ni 'Cargando…', y que la app hable el idioma del usuario."_

**Estado (2026-06-15):** sprint de pulido pre-lanzamiento que ataca tres frentes en paralelo sobre lo ya construido (S5 Misiones, S2 Streaming, S6 Retención): **(1) fricción de Misiones y Go Live**, **(2) internacionalización ES/EN completa**, y **(3) la guerra contra la "pantalla borrosa / Cargando…" en las cards del feed**. Todo desplegado a develop por bloques.

> **Nota de bitácora:** el progreso se ordena en días corridos **10 → 13 jun** + cierre el **15 jun**. El frente de **i18n** se registra el **12 jun** para reflejar la continuidad del trabajo de esos días.

---

## 🎯 Objetivo

No es un sprint de features nuevas sino de **quitar fricción y pulir** lo que ya funciona, para que GlobeLiv sea presentable al público. Tres líneas de trabajo:

| Frente | Qué resuelve |
|---|---|
| **Misiones / Go Live** | Crear y aceptar misiones sin formularios confusos; descubrir misiones aunque no haya geo; arrancar a transmitir más rápido y con la cámara correcta. |
| **i18n ES/EN** | Toda la app pasa por `t()` con catálogo ES/EN y selector de idioma. |
| **Previews del feed** | Las cards del Home muestran el video real lo antes posible, sin gradientes, blur ni "Cargando…" eternos. |

---

## 🛠 Lo que se construyó (por bloque)

### 🧭 Bloque A — Misiones y Go Live (10, 11, 13 jun)

Fricción de creación, descubrimiento y arranque de transmisión sobre el marketplace de S5.

- **Autocompletar el Go Live desde la misión** — al aceptar una misión, el título y la descripción del go-live se rellenan solos desde la misión; aceptar **solo confirma el título** en vez de re-pedir todo. `323b530`, `ba90c23`.
- **Cámara trasera por defecto** al aceptar una misión (tiene sentido: vas a mostrar un lugar, no tu cara). `ba90c23`.
- **Descubrimiento más permisivo** — el feed de "Cerca" dejaba fuera misiones sin geo y las propias; ahora **no las esconde**, y el feed muestra **TODAS las misiones activas**, no solo las cercanas (con las cercanas priorizadas). `16cbe59`, `7bd57f3`.
- **Buscador de lugares con autocompletado** al crear una misión, en vez de escribir el lugar a mano. `194a688`.
- **Geocoding sesgado a la zona del creador** para nombres ambiguos (ej. "San Miguel" hay decenas) — resuelve al más cercano al creador. `e67b39d`.
- **Mínimo para cumplir una misión: 15 → 10 min** (umbral más alcanzable). `d21f27e`.
- **Agenda:** el evento recién creado se **resalta unos segundos** para confirmar visualmente que se guardó. `8530a64`.

### 🌐 Bloque B — Internacionalización ES/EN (12 jun)

- **Base de i18n** con selector de idioma y helper `t()`; catálogo `es.json` / `en.json`. `cd41cdc`.
- **Traducción de toda la app a `t()`** — barrido completo: nav, notificaciones, perfil, settings, wallet, misiones, etc. **511 claves por idioma, 54 archivos tocados** (`apps/web/src/i18n/{es,en}.json`, `messages.ts`). `a0256a4`.

### 🎥 Bloque C — Previews del feed sin "pantalla borrosa" (10, 13, 15 jun)

El frente más largo del sprint: que la card del Home pase **directo al video real** sin gradientes de color, blur ni "Cargando…" interminable. Fue una saga iterativa (con reverts) hasta dar con un patrón estable. Estado final:

- **Dual-stream:** capa de **baja resolución** dedicada para las cards del feed (la alta queda para el player), para que el preview suba/arranque rápido sin cargar el render de la card. `edfa0da`, intento previo revertido `6d372ce`/`c92dca3`.
- **Primer frame del host subido desde el Go Live** _antes_ de navegar, para tener miniatura inmediata cuando alguien recién empieza a transmitir. `540797e`, `a974002`, `c840b1f`, `6244610`.
- **Miniatura fiable** — se probó `Image().onload`, luego `fetch()`, y se aterrizó en un `<img>`/probe `Image()` retenido con `onLoad` confiable y probe rápido (~400 ms). `3abb31a`, `e98dfbe`, `f5520d4`, `4cc24df`, `6db7950`.
- **Se eliminó el gradiente de color de la card** (era lo que el usuario percibía como "pantalla borrosa"); reemplazado por un **placeholder estilo TikTok** (avatar borroso) sin texto "Cargando…". `5175ab4`, `97e142c`, `ba8a9a9` (shimmer Skeleton intermedio).
- **Nunca pantalla negra:** el video se revela solo cuando hay un **frame pintado y NO-negro** (se mide el brillo del primer frame). `148a4b7`, `7cba5eb`, `ea48dd3`.
- **Auto-actualizador:** la web **se recarga sola cuando hay un deploy nuevo** (evita que el usuario quede en una versión vieja durante la iteración rápida). `d9b1962`.

---

## 🔬 Verificación

- `typecheck` / `lint` / `build` verdes por bloque antes de cada push (criterio de siempre).
- Las traducciones se cargaron en `tsconfig.json` (resolución de JSON) — build de web pasa con el catálogo nuevo.
- El bloque C se validó en vivo en develop iterando con el auto-actualizador; varios commits son reverts/ajustes de la misma sesión hasta el patrón estable.

---

## 🚀 Deploy (✅ a develop)

- Dual-push por bloque a `danii-bot/globeliv-app` (Railway/API) + `AdminGlobeliv/Globeliv` (Vercel/web).
- Último commit del rango: `edfa0da` (dual-stream para cards), 15 jun.

---

## 🧾 Deuda técnica / pendientes

- **Saga de previews con ruido en el historial:** muchos commits del 15 jun son iteraciones/reverts sobre el mismo problema; conviene consolidar la lógica de revelado de video (brillo + probe + dual-stream) en un solo hook documentado.
- **i18n incompleto en strings nuevas:** cualquier feature nueva debe entrar ya por `t()`; faltan auditar strings hardcodeadas que pudieran haber quedado fuera del barrido.
- **Auto-actualizador** es pragmático para develop; revisar si debe quedar en producción o gatillarse distinto.
- Siguen abiertos de sprints previos: **S4 pagos** (proveedor para Perú), **mocks de S6** (FCM/recording/Windy) y **Parte B de S7** (M5 NSFW + M6 alertas) esperando credenciales.

---

## 🔗 Conexiones

- Viene de: [[Sprint 7 - Moderación (8 jun)]]
- Construye sobre: [[Sprint 5 - Misiones (2-3 jun)]], [[Sprint 2 - Streaming Core (27-29 may)]], [[Sprint 6 - Retención y Pasaporte (3-5 jun)]]
- Estado en memoria: [[globeliv_feed_live_preview_decision]], [[globeliv_no_blur_loading_streaming]]
- Próximo: cerrar **Parte B de S7** con creds y avanzar a **lanzamiento**.
