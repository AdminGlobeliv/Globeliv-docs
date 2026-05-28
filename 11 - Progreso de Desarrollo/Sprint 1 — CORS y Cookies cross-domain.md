---
sprint: 1
fecha: 2026-05-27
tema: cors-cookies
---

# Sprint 1 — CORS y Cookies cross-domain

> El bug que más tiempo robó del sprint. Tres iteraciones hasta que `globeliv.com` (Vercel) y `api.globeliv.com` (Railway) pudieron compartir sesión sin que Chrome la bloqueara.

Sprint maestro: [[Sprint 1 - Auth y Usuarios (26-27 may)]]

---

## 🧨 El problema

Tras el primer deploy a producción:

1. Usuario hace signup en `https://www.globeliv.com` → 200 OK
2. API setea cookie `globeliv_access`
3. Cliente navega a `/profile` → tRPC `auth.me`
4. **Falla**: cookie no llega al request → 401 → redirect a `/signin`
5. Ciclo infinito

## 🧪 Diagnóstico

### Causa 1: CORS allowlist demasiado estricta (`60c04cd`)

CORS solo permitía `https://globeliv.com` hardcoded. Pero el frontend Vercel servía desde:
- `https://globeliv-xyz123.vercel.app` (preview deploys)
- `https://www.globeliv.com` (root domain redirige 307 a www)

**Fix:** allowlist por **función** que combina origins estáticos + regex de subdominios:

```ts
const staticOrigins = env.NODE_ENV === "production"
  ? [env.WEB_PUBLIC_URL]
  : ["http://localhost:3000", "http://127.0.0.1:3000", env.WEB_PUBLIC_URL];

const globelivProductionPattern = /^https:\/\/([\w-]+\.)?globeliv\.com$/u;
const vercelPreviewPattern = /^https:\/\/[\w-]+\.vercel\.app$/u;

const isAllowedOrigin = (origin: string): boolean => {
  if (staticOrigins.includes(origin)) return true;
  if (env.NODE_ENV !== "production") return false;
  return globelivProductionPattern.test(origin) || vercelPreviewPattern.test(origin);
};
```

### Causa 2: `globeliv.com` → `www.globeliv.com` rompe regex inicial (`9533829`)

La primera versión solo permitía `globeliv.com` exacto. El redirect 307 hacía que el browser intentara CORS preflight desde `https://www.globeliv.com` y fallaba. Fix: regex `/^https:\/\/([\w-]+\.)?globeliv\.com$/u` que cubre `globeliv.com`, `www.`, `admin.`, `api.`, etc.

### Causa 3: Chrome 2024+ bloquea third-party cookies (`491ad23`)

API en `globeliv-production.up.railway.app` (Railway default) y frontend en `*.vercel.app` → distintos eTLD+1 → **third-party cookie**. Chrome la descartaba aunque CORS permitiera el origin.

**Fix temporal:** `Partitioned: true` (CHIPS — Cookies Having Independent Partitioned State):

```ts
// Solo en producción cuando API y web están en distintos eTLD+1
{
  httpOnly: true,
  secure: true,
  sameSite: "none",
  partitioned: true,  // ← CHIPS
  path: "/",
}
```

Esto **funcionó** pero introdujo otro bug: durante OAuth Google, el callback en `api.globeliv.com` y el `/profile` en `globeliv.com` recibían cookies en **particiones distintas** → /profile no encontraba la sesión recién creada.

### Causa 4: La solución real (`47cac4c`) — same eTLD+1

Configurar el subdominio `api.globeliv.com` (Railway custom domain) → frontend y API comparten `globeliv.com` como eTLD+1 → cookies **dejan de ser third-party**.

**Estado final:**

```ts
export function accessCookieOptions() {
  const env = loadEnv();
  const isProd = env.NODE_ENV === "production";
  return {
    httpOnly: true,
    secure: isProd,
    sameSite: "lax" as const,   // ← era "none" + partitioned
    path: "/",
    maxAge: env.JWT_ACCESS_TTL_SECONDS * 1000,
    // domain implícito → cookie se aplica a api.globeliv.com
    // pero el browser la envía también a www.globeliv.com porque
    // share el eTLD+1 globeliv.com... pero solo si el sameSite es 'lax'
  };
}
```

> Aclaración técnica: la cookie de `api.globeliv.com` con `sameSite: lax` y dominio implícito **no se envía** automáticamente desde `www.globeliv.com` en requests XHR. **Pero** el frontend hace `credentials: 'include'` y como **eTLD+1 es el mismo**, ya no es cross-site → Chrome la incluye.

---

## 📊 Resumen iteraciones

| Iteración | Commit | Estado cookies | Problema resuelto / introducido |
|---|---|---|---|
| Inicial | `cbf268a` | `sameSite: none`, `secure: true` | OK en localhost, falla en prod |
| Allowlist CORS | `60c04cd` | igual | CORS pasa, cookies aún bloqueadas |
| Regex subdominios | `9533829` | igual | `www.` ya pasa CORS, cookies igual |
| CHIPS partitioned | `491ad23` | `partitioned: true` | Cookies llegan, **rompe OAuth** |
| Subdominio API | `47cac4c` | `sameSite: lax`, sin partitioned | ✅ Funciona, OAuth funciona |

---

## 🧠 Lección general

> **Mismo eTLD+1 entre frontend y API simplifica TODO el modelo de cookies.** Si vas a deploy frontend y backend separados, configura subdominios del mismo root domain desde el día 1 — evita 2 commits de fixes y un sistema de cookies "exótico".

Esta lección está implícita en el commit `47cac4c` y debería entrar al [[ROADMAP]] como guideline durable.

---

## 🔗 Archivos involucrados

- `Globeliv/apps/api/src/main.ts` — CORS middleware con `isAllowedOrigin`
- `Globeliv/apps/api/src/lib/jwt.ts` — `accessCookieOptions()`
- `Globeliv/apps/api/src/lib/oauth.ts` — `oauthTempCookieOptions()`
- `Globeliv/apps/web/src/lib/trpc.ts` — `credentials: 'include'` en tRPC client

---

## 🔍 Test rápido tras cambios de cookies

Para verificar que un cambio de cookies no rompió la sesión:

```bash
# 1. Curl signin → guardar cookie
curl -c cookies.txt -X POST https://api.globeliv.com/trpc/auth.signin \
  -H "Content-Type: application/json" -H "Origin: https://www.globeliv.com" \
  -d '{"email":"...","password":"..."}'

# 2. Inspeccionar
cat cookies.txt

# 3. Llamar /auth.me con la cookie
curl -b cookies.txt https://api.globeliv.com/trpc/auth.me \
  -H "Origin: https://www.globeliv.com"
```

Si devuelve `{ user: { … } }` → todo OK.
