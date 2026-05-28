---
sprint: 1
fecha: 2026-05-27
tema: frontend
---

# Sprint 1 — Profile, Settings, OAuth y Avatar

> Lo que el stakeholder ve cuando entra a `globeliv.com` tras crear cuenta. Pantallas implementadas + Google OAuth + uploader de avatar con resize client-side.

Sprint maestro: [[Sprint 1 - Auth y Usuarios (26-27 may)]]

---

## 📱 Pantallas implementadas (en `apps/web/src/app/`)

| Ruta | Archivo | Función |
|---|---|---|
| `/welcome` | `welcome/page.tsx` | Landing pública placeholder (CTA "Empezar") |
| `/signin` | `signin/page.tsx` | Login con email + password + botón Google |
| `/login` | `login/page.tsx` | Signup con auto-login + botón Google |
| `/profile` | `profile/page.tsx` | Mi perfil — header con avatar + stats + edit |
| `/settings` | `settings/page.tsx` | 5 grupos: Cuenta, Streaming, Notificaciones, Pagos, Privacidad |
| `/u/[username]` | `u/[username]/page.tsx` | Perfil público de otro usuario |

Cubre las pantallas del spec:
- [[Pantalla 6 - Perfil]] — completa
- [[Pantalla 7 - Configuración]] — UI completa, persistencia parcial

---

## 🧩 Componentes nuevos en `apps/web/src/components/`

### `auth/`
- `auth-shell.tsx` — layout compartido por `/signin` y `/login` (centered card sobre `BgDecoration`)
- `form-field.tsx` — input genérico con `react-hook-form` register + error state
- `oauth-button.tsx` — botón con ícono que dispara `window.location = '/auth/google/start'`
- `brand-icons.tsx` — íconos Google/Apple/etc. SVG inline
- `avatar.tsx` — `<img>` con fallback a iniciales
- `auth-motion.ts` — variantes Framer Motion compartidas

### `profile/`
- `profile-shell.tsx` — layout mobile single-col / desktop split
- `profile-header.tsx` — banner con avatar grande + display name + username + edit
- `profile-edit-form.tsx` — formulario inline para `displayName`, `username`, `location`
- `profile-cards.tsx` — stats grid (streams, ratings — placeholders hasta Sprint 2+)
- `avatar-uploader.tsx` — file input → canvas resize → data URL → tRPC `auth.setAvatar`

### `settings/`
- `settings-group.tsx` — grupo con título y lista de items
- `settings-item.tsx` — fila estándar (label + helper text + control)
- `settings-toggle.tsx` — switch custom (workaround Tailwind 4 + HeroUI)
- `change-password-card.tsx` — modal flow para `auth.changePassword`
- `delete-account-card.tsx` — confirm modal con password gate → `auth.deleteAccount`

### `nav/`
- `bottom-nav.tsx` — mobile bottom nav (Home, Buscar, Go Live, Bandeja, Yo)
- `top-nav.tsx` — desktop top nav

### `shared/`
- `bg-decoration.tsx` — gradiente decorativo de marca (usado en /signin y /profile)

---

## 🎨 Design system "Kinetic v1"

Pivot del Sprint 0 (paleta básica) a una paleta dark-first con marca diferenciada.

**Archivo:** `Globeliv/packages/ui/src/tokens.css`

### Paleta

| Token | Valor | Uso |
|---|---|---|
| `--color-brand-red` | `#ff3b4e` | Color "Liv" + estados live + CTA primario |
| `--color-brand-red-soft` | `#ffe7ea` | Hover states light |
| `--color-brand-blue` | `#3b82f6` | Acentos secundarios |
| `--color-brand-green` | `#22c55e` | Success states |
| `--color-brand-gold` | `#f59e0b` | Highlights (rating, badges) |
| `--color-brand-surface` | `#0a0e1a` | Fondo principal dark |
| `--color-brand-surface-elevated` | `#141828` | Cards sobre fondo |
| `--color-brand-surface-deep` | `#050810` | Hero sections |
| `--color-brand-surface-tint` | `#0f1430` | Subtle layers |
| `--color-brand-white` | `#f1f5f9` | Texto principal sobre dark |
| `--color-brand-text-secondary` | `#94a3b8` | Texto helper |
| `--color-brand-text-muted` | `#64748b` | Placeholders |

### Tipografía

- `--font-sans` → **Inter** (body, inputs, labels) — vía `next/font/google`
- `--font-display` → **Space Grotesk** (títulos, logo "GlobeLiv") — vía `next/font/google`
- `--font-mono` → JetBrains Mono (chips, IDs futuros)

**Cambio vs Sprint 0:** se agregó Space Grotesk para títulos. Sprint 0 solo tenía Inter + JetBrains.

### Radios y sombras

- `--radius-sm: 8px` / `md: 12px` / `lg: 16px` / `pill: 9999px`
- `--shadow-glow-brand: 0 4px 20px rgba(255, 59, 78, 0.3)` — glow rojo para CTAs en hero

### Compatibilidad con [[Reglas de UX que no se negocian]]

- Regla #1 (cero registros forzados al inicio): `/welcome` muestra valor antes de pedir signup
- Regla #2 (sin friction visual): paleta dark con UN solo color de marca, hierarchy clara

---

## 🌐 Google OAuth (arctic)

**Archivos:**
- `Globeliv/apps/api/src/lib/oauth.ts` — cliente arctic + cookies temp
- `Globeliv/apps/api/src/modules/oauth/google-oauth.controller.ts` — endpoints HTTP

### Flow

```
1. User → click "Continuar con Google"
2. Browser GET /auth/google/start (en api.globeliv.com)
   → seteo cookies temp: g_oauth_state, g_oauth_verifier (10 min, httpOnly, lax)
   → 302 a accounts.google.com con state + PKCE
3. User autoriza en Google
4. Google → GET /auth/google/callback?code=…&state=…
   → verify state coincide con cookie
   → exchange code por access_token (PKCE verifier)
   → fetch userinfo (sub, email, name, picture)
5. Decisión de account linking:
   a. Hay user con google_id = sub → login
   b. No, pero hay user con lower(email) = email → LINK (set google_id)
   c. Ninguno → crear user nuevo (password_hash = null)
6. signAccessToken + set cookie globeliv_access
7. 302 a /profile
```

### Decisiones

- **Endpoints REST clásicos, no tRPC** — OAuth funciona con redirects top-level, no con AJAX
- **PKCE habilitado** (S256) — arctic lo gestiona internamente
- **Lazy client** — el API arranca sin `GOOGLE_CLIENT_*` (los endpoints devuelven 503)
- **Account linking automático por email** — si un user creó cuenta con email/password y luego entra con Google al mismo email, se linkea. Razón: UX más suave que forzar "tu email ya existe, signin primero".
- **`password_hash` nullable** — los users solo-OAuth no tienen password configurada hasta que la creen en /settings

### Configuración en Google Cloud Console

- OAuth Client ID tipo "Web application"
- Authorized redirect URI: `https://api.globeliv.com/auth/google/callback`
- Authorized JavaScript origins: `https://www.globeliv.com`

---

## 🖼 Avatar uploader

**Archivo:** `Globeliv/apps/web/src/components/profile/avatar-uploader.tsx`

### Pipeline

```
1. <input type="file" accept="image/*"> dispara onChange
2. FileReader → load como Image()
3. Canvas:
   - cover-fit a 256x256
   - drawImage()
4. canvas.toDataURL('image/jpeg', 0.85) → data URL
5. Validar size <= 150KB (si excede: reducir quality y reintentar)
6. tRPC mutation auth.setAvatar({ dataUrl })
7. Backend valida re-tamaño y guarda en columna users.avatar_url
```

### Por qué data URL en DB (decisión temporal)

- **Pros:** zero infra extra, transactional con el resto del row, sin signed URL flow
- **Cons:** row weight más alto (~150KB), no CDN, lectura tiene que pasar por API

**Migration plan:** cuando se active billing de Cloudflare R2 (probable Sprint 3-4):
1. `pnpm --filter @globeliv/database db:generate` agregando columna `avatar_r2_key`
2. Backfill batch: data URL → upload R2 → set `avatar_r2_key`
3. Drop `avatar_url` o mantenerlo como CDN URL

---

## ⚠️ Workaround Tailwind 4 + HeroUI + pnpm symlinks

**Síntoma:** `@source` directive de Tailwind 4 no encuentra los class names dentro de `@heroui/react` porque pnpm los enlaza simbólicamente desde `node_modules/.pnpm/`.

**Workaround actual:** algunos componentes custom (`TextInput`, `SettingsItem`, `SettingsToggle`) en lugar de los HeroUI equivalentes. Cumple visualmente pero **rompe la regla** [[heroui_first]].

**Fix definitivo pendiente** (registrado en deuda técnica de [[Sprint 1 - Auth y Usuarios (26-27 may)]]):
- Opción A: agregar `@source "../../node_modules/@heroui/react/dist/**/*.js"` (frágil)
- Opción B: configurar `safelist` con patrones de HeroUI
- Opción C: usar `tailwindcss/postcss` con el plugin oficial de HeroUI cuando salga compatible

---

## 🔗 Archivos clave

- `Globeliv/apps/web/src/lib/trpc.ts` — tRPC client con `credentials: 'include'` (cross-origin cookies)
- `Globeliv/apps/web/src/lib/auth/next-param.ts` — helper para `?next=` redirect
- `Globeliv/apps/web/src/app/layout.tsx` — root layout con providers
- `Globeliv/apps/web/src/app/providers.tsx` — HeroUIProvider + QueryClient + tRPC provider
- `Globeliv/packages/ui/src/tokens.css` — design tokens completos
