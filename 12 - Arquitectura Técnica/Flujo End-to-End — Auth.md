---
sección: arquitectura
tema: flujo-auth
---

# Flujo End-to-End — Auth

> Las **3 secuencias críticas** de autenticación con diagramas paso a paso: signup (con auto-login), signin, Google OAuth. La 4ª (logout) es trivial — clear cookie.

---

## 🔐 Secuencia 1 — Signup (email + password)

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario (browser)
    participant W as www.globeliv.com<br/>(Next /login)
    participant A as api.globeliv.com<br/>(/trpc/auth.signup)
    participant DB as Postgres

    U->>W: Llena form (email, password, displayName)
    W->>W: react-hook-form + Zod valida client-side
    W->>A: POST /trpc/auth.signup<br/>credentials: include
    A->>A: signupInput.parse() Zod re-validate
    A->>DB: SELECT WHERE lower(email) = ?
    DB-->>A: 0 rows (no existe)
    A->>A: bcrypt.hash(password, cost=10)
    A->>DB: INSERT users RETURNING { id, email, displayName, ... }
    DB-->>A: user creado
    A->>A: signAccessToken(user.id)<br/>HS256, iss/aud/exp claims
    A-->>W: 200 + Set-Cookie globeliv_access<br/>HttpOnly; Secure; SameSite=Lax
    W->>W: TanStack invalida cache auth.me
    W->>U: router.push('/profile')
```

### Casos de error mapeados

| Situación | Status / Code | UI mostrará |
|---|---|---|
| Email ya existe | `CONFLICT` | "Este correo ya está registrado" |
| Email/password malformados (Zod) | `BAD_REQUEST` | Mensaje del field error |
| DB caída | `INTERNAL_SERVER_ERROR` | "No se pudo crear el usuario" |

---

## 🔐 Secuencia 2 — Signin (email + password)

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant W as www.globeliv.com<br/>(/signin)
    participant A as api.globeliv.com<br/>(/trpc/auth.signin)
    participant DB as Postgres

    U->>W: Llena email + password
    W->>A: POST /trpc/auth.signin
    A->>DB: SELECT user (incluye password_hash, deleted_at)
    alt usuario no existe / borrado / solo-OAuth
        A->>A: bcrypt.compare(input, DUMMY_HASH)<br/>(tiempo constante)
        A-->>W: 401 UNAUTHORIZED<br/>"Correo o contraseña incorrectos"
    else password incorrecto
        A->>A: bcrypt.compare(input, candidate.passwordHash)
        A-->>W: 401 UNAUTHORIZED (mismo mensaje)
    else password correcto
        A->>A: signAccessToken(user.id)
        A-->>W: 200 + Set-Cookie + { user }
        W->>U: router.push('/profile')
    end
```

### Por qué el DUMMY_HASH

User enumeration: si "user no existe" responde rápido y "password incorrecto" responde lento (bcrypt tarda ~80ms), un atacante puede deducir qué emails existen.

**Fix:** cuando el user no existe, igualmente corremos `bcrypt.compare(input, DUMMY_HASH)` → mismo tiempo de respuesta.

Detalle: [[Sprint 1 — Sistema de Auth]] sección "bcryptjs — hashing".

---

## 🔐 Secuencia 3 — Google OAuth con account linking

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant W as www.globeliv.com
    participant A as api.globeliv.com<br/>(/auth/google/*)
    participant G as accounts.google.com
    participant DB as Postgres

    U->>W: Click "Continuar con Google"
    W->>U: window.location = api.globeliv.com/auth/google/start
    U->>A: GET /auth/google/start
    A->>A: generateState() + generateCodeVerifier()
    A-->>U: Set-Cookie g_oauth_state, g_oauth_verifier<br/>(httpOnly, lax, 10min)<br/>302 → G con state + PKCE
    U->>G: GET accounts.google.com/oauth/auth
    U->>G: Autoriza
    G-->>U: 302 → api/auth/google/callback?code=...&state=...
    U->>A: GET /auth/google/callback (lleva cookies)
    A->>A: Verify state == cookie.state
    A->>G: POST /token (exchange code + PKCE verifier)
    G-->>A: { access_token }
    A->>G: GET /userinfo
    G-->>A: { sub, email, name, picture }
    A->>DB: SELECT WHERE google_id = sub
    alt user con google_id existe
        DB-->>A: user
    else else SELECT WHERE lower(email) = email
        DB-->>A: user existente<br/>(creado antes con email/pwd)
        A->>DB: UPDATE SET google_id = sub<br/>(LINK)
    else else nuevo user
        A->>DB: INSERT users (email, google_id, displayName, ...)<br/>password_hash = NULL
    end
    A->>A: signAccessToken(user.id)
    A-->>U: Set-Cookie globeliv_access<br/>302 → /profile
```

### 3 ramas de decisión

| Caso | Acción | Resultado |
|---|---|---|
| `google_id` ya existe | Login directo | Token emitido |
| Email ya existe (otro signup) | **LINK** — set `google_id` en el row existente | Mismo user, ahora con dos métodos de login |
| Ninguno | Crear nuevo user con `password_hash = NULL` | User solo-OAuth |

> Por qué el linking automático: UX más suave. Si el user creó cuenta con email y luego entra con Google al mismo email, no le forzamos "ya existe, signin primero". Riesgo: si un atacante crea cuenta con email de víctima y la víctima luego entra con Google → atacante perdió acceso (no tiene el OAuth de Google). **El email verificado de Google es la prueba de identidad.**

---

## 🛡 Verificación en cada request autenticado

Después del signup/signin/OAuth, **cada llamada protegida** sigue este patrón:

```mermaid
sequenceDiagram
    autonumber
    participant W as www.globeliv.com
    participant A as api.globeliv.com
    participant DB as Postgres

    W->>A: POST /trpc/users.updateProfile<br/>Cookie: globeliv_access=...
    A->>A: cookie-parser → req.cookies
    A->>A: protectedProcedure middleware:<br/>verifyAccessToken(jose)
    alt token válido
        A->>A: ctx.userId = payload.sub
        A->>DB: UPDATE users WHERE id = ctx.userId
        A-->>W: 200 + updated user
    else token inválido / expirado / ausente
        A-->>W: 401 UNAUTHORIZED
        W->>W: tRPC error.data.code === 'UNAUTHORIZED'
        W->>W: router.replace('/signin?next=...')
    end
```

---

## 🍪 Por qué la cookie funciona cross-domain

`www.globeliv.com` (Vercel) y `api.globeliv.com` (Railway) **comparten eTLD+1** = `globeliv.com`. Con eso:

- `sameSite: 'lax'` permite que la cookie viaje en navegaciones top-level + XHR a otros subdominios del mismo eTLD+1
- `credentials: 'include'` en el cliente fetch la incluye en CORS requests
- CORS del API tiene `Access-Control-Allow-Credentials: true` + `Access-Control-Allow-Origin: <origen exacto>` (no `*`)

> Si frontend y API estuvieran en **eTLD+1 distintos** (ej. `globeliv.com` + `globeliv-api.com`), tendríamos que volver a `sameSite: 'none' + partitioned: true` (CHIPS) — esto fue el bug del Sprint 1. Ver [[Sprint 1 — CORS y Cookies cross-domain]].

---

## 🔗 Notas relacionadas

- [[Seguridad y Auth]] — detalles de JWT, bcrypt, cookies (referencia técnica)
- [[Sprint 1 — Sistema de Auth]] — implementación con código
- [[Sprint 1 — Profile, Settings y OAuth]] — UI del OAuth
- [[Sprint 1 — CORS y Cookies cross-domain]] — historia del bug
- [[Modelo de Datos]] — schema `users` con `google_id`, `password_hash` nullable
