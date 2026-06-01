---
sprint: 2
nombre: Cierre y fixes
fecha: 2026-05-29
estado: cerrado
tema: integración Agora real + deploy git-driven + pulido UI/UX
---

# Sprint 2 — Cierre y fixes (29 may) ✅

> Jornada que **cerró el Sprint 2**: pasamos de "todo en mock, sin pushear" a **Agora real corriendo end-to-end en `dev.globeliv.com`** con auto-deploy desde git, más una tanda grande de pulido de UI/UX sobre Go Live y el Player. Nota maestra: [[Sprint 2 - Streaming Core (27-29 may)]].

---

## 🎯 Qué se resolvió

Tres frentes en un día:

1. **Infra de deploy** — que `git push develop` despliegue web + API solos.
2. **Agora real** — cambiar el mock por video real bidireccional en develop.
3. **UI/UX** — una lista larga de fixes visuales y de flujo en Go Live y `/watch`.

---

## 🚀 1. Deploy git-driven a develop

El problema de raíz: los previews de Vercel por CLI generaban una URL-hash distinta en cada deploy, y Railway tenía clavado un preview viejo → rompía OAuth/cookies en cada deploy. Detalle completo del setup de dominios y repos en [[globeliv-dev-environment-setup]] (memoria).

- **Dominios estables:** web develop → `dev.globeliv.com` (Vercel), API develop → `api-dev.globeliv.com` (Railway). Ambos bajo `globeliv.com` para que las cookies sean `SameSite=lax` y el login persista.
- **Repo propio `danii-bot/globeliv-app`** — `AdminGlobeliv/Globeliv` no es del usuario (danii-bot solo era colaborador) → no se podía instalar la Vercel GitHub App. Se creó repo privado propio, se migró el historial (main/develop/backup) y se conectó a Vercel + Railway.
- **Resultado:** un solo `git push origin develop` → auto-deploy de web (Vercel) **y** API (Railway env develop). Verificado: push → `dev.globeliv.com` se actualiza solo con `api-dev` horneado.

Commits: `becda5b` (cookies sameSite=none+partitioned en previews), `e6a52be` (retrigger Railway), `3af7ef0` (primer deploy git-driven a dev.globeliv.com).

> ⚠️ Deuda: prod `NEXT_PUBLIC_API_URL` sigue vacío → si se redespliega prod por git, el front de prod apunta al API de develop por el fallback. Arreglar cuando se toque prod.

---

## 🎥 2. Agora real en develop

- **App ID `52dcf7d5b3074111b4e04cc74a31b6f8`** configurado.
- **Railway develop:** `AGORA_USE_MOCK=false` + `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE`.
- **Vercel Preview:** `NEXT_PUBLIC_AGORA_APP_ID` + `NEXT_PUBLIC_AGORA_USE_MOCK=false`.
- **Verificado:** `streams.create` emite token real (prefijo `007…`) y el viewer ve el video real del host.

Commit: `9f2df9c` (`ci: rebuild con credenciales Agora reales (NEXT_PUBLIC_AGORA_APP_ID)`).

> ⚠️ **Producción NO tiene Agora.** Los env de Production (Railway + Vercel) están sin setear — si se quiere streaming en prod, repetir este setup ahí.

---

## 🎨 3. Pulido UI/UX (Go Live + Watch)

Tanda de fixes sobre la experiencia de transmitir y ver, en orden cronológico:

### Go Live (`/go-live`)
- `0dd4090` — **fix botón "Iniciar transmisión":** el gate (`cameraReady && título válido && ubicación no vacía`) deshabilitaba el botón sin avisar. Ahora muestra un **hint de los requisitos faltantes** en [go-live-form.tsx](Globeliv/apps/web/src/app/go-live/_components/go-live-form.tsx).
- `f1cfef2` / `3f8503e` — **labels fuera del input** (sin solape con el placeholder) + monetización en vertical.
- `56ef3a5` — inputs más visibles (fondo + borde 2px + foco rojo) y ejemplos en placeholder.
- `e81aa3b` / `38ff400` — **monetización con botones-tarjeta** (se ocultó el círculo gigante del radio que se veía encimado).

### Watch / Player (`/watch/[id]`)
- `03361ff` — **pantalla de fin de transmisión** dinámica tipo TikTok.
- `6577c37` — **host full-screen tipo TikTok** y cámara grande para el viewer.
- `801ef8d` — **gestionar stream activo** (reanudar/terminar) + **preview casi-en-vivo en el feed**.
- `2d815f6` — **perf del feed:** preview solo en cards visibles (IntersectionObserver) + pausa de captura del host en background.
- `3744c6e` / `84ce281` / `d056d58` — **modal de confirmación al terminar:** la X del host ya no sale dejando el stream colgado; abre un modal (ícono power, botones apilados full-width). Se iteró el diseño y se quitó el checkbox/X redundantes.

### Global
- `6bc8199` — **tema dark de HeroUI activado** (clase `dark` en `html`) — arregla texto oscuro sobre botones oscuros. Respeta la regla [[heroui-first]].
- `5cc1f6a` — **espejar solo la cámara frontal** (selfie natural); la trasera en orientación real.

---

## 🐛 Bugs de sesión detectados (mitigados, no resueltos del todo)

- **Menú sin Perfil/Configuración tras un rato = sesión expirada.** No hay refresh token (solo la cookie `globeliv_access`, antes 15 min). **Mitigación:** `JWT_ACCESS_TTL_SECONDS` subido a 86400 (24h) en Railway develop.
- **Proper fix pendiente** (heredado a Sprint 3+): implementar refresh tokens (cookie refresh + endpoint `auth.refresh` + auto-renovación al 401). El bottom-nav mobile tampoco tiene entrada "Configuración".

---

## 🔗 Conexiones

- Nota maestra: [[Sprint 2 - Streaming Core (27-29 may)]]
- Sigue en: [[Sprint 3 - Realtime (30 may)]]
- Infra de dominios/repos: [[globeliv-dev-environment-setup]] (memoria)
- Estado de streaming: [[globeliv-s2-streaming-state]] (memoria)
