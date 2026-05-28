---
sprint: 1
fecha: 2026-05-27
tema: auth
---

# Sprint 1 — Sistema de Auth

> JWT en cookie httpOnly + bcrypt. Todo el lifecycle de cuenta (signup, signin, logout, me, updateProfile, changePassword, deleteAccount) implementado.

Sprint maestro: [[Sprint 1 - Auth y Usuarios (26-27 may)]]

---

## 🧠 Modelo conceptual

```
Browser                    api.globeliv.com           globeliv (DB)
   │                              │                        │
   │ POST /trpc/auth.signup       │                        │
   ├─────────────────────────────►│                        │
   │   { email, password,         │  bcrypt.hash(12)       │
   │     displayName }            │  INSERT user           │
   │                              ├───────────────────────►│
   │                              │  signAccessToken(uid)  │
   │                              │  set-cookie HttpOnly   │
   │                              │  { Secure, SameSite }  │
   │ ◄────────────────────────────┤                        │
   │   { user }                   │                        │
   │                              │                        │
   │ GET /profile (Next SSR)      │                        │
   │   cookie: globeliv_access=…  │                        │
   │ POST /trpc/auth.me           │                        │
   ├─────────────────────────────►│                        │
   │                              │  jwtVerify(cookie)     │
   │                              │  SELECT user           │
   │                              ├───────────────────────►│
   │ ◄────────────────────────────┤  { user }              │
```

---

## 🔐 JWT — implementación

**Archivo:** `Globeliv/apps/api/src/lib/jwt.ts`

- Librería: **`jose`** (no `jsonwebtoken` por compatibilidad ESM)
- Algoritmo: **HS256** con `JWT_SECRET` (32+ chars)
- Claims:
  - `iss` = `env.JWT_ISSUER` (`globeliv.com`)
  - `aud` = `env.JWT_AUDIENCE` (`globeliv-web`)
  - `sub` = `userId` (UUID)
  - `iat`, `exp` con TTL configurable (`JWT_ACCESS_TTL_SECONDS`)

**Cookie:**
- Nombre: `globeliv_access`
- `httpOnly: true`, `secure: production`, `sameSite: 'lax'`, `path: '/'`
- Domain implícito (api.globeliv.com) → cookies se comparten en eTLD+1 `globeliv.com`

> Sprint 6+ migrará a **RS256** si necesitamos verificar tokens fuera del API. Sprint 2 trae refresh tokens con `jti` + Redis allowlist.

---

## 🔒 bcryptjs — hashing

- **Cost = 10** (~80ms en hardware moderno) — bajo el umbral de 200ms del §11.1 de `CLAUDE.md`
- Subir a 12 cuando haya hardware dedicado para auth (no compartido con tRPC general)
- **Hash dummy `$2a$10$invalidsalt…`** en signin cuando user no existe o es solo-OAuth → mantiene **tiempo constante** y evita user enumeration

```ts
const DUMMY_HASH = "$2a$10$invalidsaltinvalidsalt22fakefakefakefakefakefakefakefa";
if (!candidate || candidate.deletedAt || !candidate.passwordHash) {
  await bcrypt.compare(input.password, DUMMY_HASH);  // ← bait
  throw new TRPCError({ code: "UNAUTHORIZED", message: "Correo o contraseña incorrectos" });
}
```

---

## 📜 Procedures tRPC implementados

**Archivo:** `Globeliv/apps/api/src/trpc/routers/auth.router.ts`

| Procedure | Tipo | Función |
|---|---|---|
| `auth.signup` | mutation, public | Crear cuenta + auto-login (set cookie) |
| `auth.signin` | mutation, public | Validar credenciales + set cookie |
| `auth.logout` | mutation, public | Clear cookie |
| `auth.me` | query, **protected** | Devuelve `user` del cookie actual |
| `auth.updateProfile` | mutation, **protected** | Editar `displayName`, `username`, `location` |
| `auth.changePassword` | mutation, **protected** | Old + new password, bcrypt |
| `auth.setAvatar` | mutation, **protected** | Acepta data URL (cap 150KB) |
| `auth.deleteAccount` | mutation, **protected** | Soft delete (`deleted_at = now()`) tras confirmar password |

### Validación de inputs

Todos los inputs son `zod` schemas definidos en `@globeliv/zod-schemas`:

- `signupInput`, `signinInput`, `updateProfileInput`, `changePasswordInput`, `setAvatarInput`, `deleteAccountInput`

Beneficio: el frontend (`apps/web`) usa los mismos schemas en `react-hook-form` → un solo lugar para reglas de validación.

---

## 🛡 `protectedProcedure` middleware

**Archivo:** `Globeliv/apps/api/src/trpc/trpc.ts`

```ts
const isAuthenticated = middleware(async ({ ctx, next }) => {
  const token = ctx.req.cookies[ACCESS_COOKIE_NAME];
  if (!token) throw new TRPCError({ code: "UNAUTHORIZED" });

  const payload = await verifyAccessToken(token);
  if (!payload) throw new TRPCError({ code: "UNAUTHORIZED" });

  return next({ ctx: { ...ctx, userId: payload.userId } });
});

export const protectedProcedure = publicProcedure.use(isAuthenticated);
```

El `ctx.userId` queda **garantizado no-null** en todo procedure que extiende `protectedProcedure` — la regla multi-tenant del CLAUDE.md (§8) usa este `ctx.userId` en cada `WHERE`.

---

## ⚠️ Workaround NestJS 11 + Express 4

**Archivo:** `Globeliv/apps/api/src/main.ts`

```ts
// ExpressAdapter.isMiddlewareApplied accede a `app.router` para detectar
// si un body parser ya está montado, pero Express 4 deprecó ese getter.
// Re-implementamos a "siempre false" → Nest aplica los parsers (idempotentes).
(ExpressAdapter.prototype as unknown as { isMiddlewareApplied: () => boolean })
  .isMiddlewareApplied = () => false;
```

**Quitar cuando:** Nest 11 soporte Express 4 nativamente o subamos a Express 5.

---

## 🧪 DB singleton compartido

Tanto Nest DI como el context tRPC necesitan la misma instancia de Drizzle DB. Sin singleton se abren **dos pools** (uno por consumidor) → desperdicia conexiones.

**Solución** en `Globeliv/apps/api/src/shared/database/instance.ts`:

```ts
let cached: ReturnType<typeof createDb> | null = null;
export function getDatabaseInstance() {
  cached ??= createDb(env.DATABASE_URL);
  return cached;
}
```

Nest DI lo inyecta vía `useFactory` y el `createTrpcContext` también lo lee de esta función → un solo pool.

---

## 🗄 Migración de schema asociada

**`0002_amazing_whistler.sql`** — make password_hash nullable (para users solo-OAuth), agrega `google_id` + unique index. Detalle en [[Sprint 0 — DB y Migraciones]].

---

## 🔗 Referencias

- `Globeliv/apps/api/src/lib/jwt.ts` — sign + verify
- `Globeliv/apps/api/src/trpc/routers/auth.router.ts` — procedures
- `Globeliv/apps/api/src/trpc/trpc.ts` — `protectedProcedure` + middleware
- `Globeliv/apps/api/src/trpc/context.ts` — context con `userId`, `db`, `res`, `req`
- `Globeliv/apps/api/src/shared/database/instance.ts` — DB singleton
- `Globeliv/packages/zod-schemas/src/users.ts` — schemas Zod de input
