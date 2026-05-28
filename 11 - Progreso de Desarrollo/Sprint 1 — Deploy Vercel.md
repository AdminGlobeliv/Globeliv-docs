---
sprint: 1
fecha_inicio: 2026-05-26
fecha_fin: 2026-05-26
tema: deploy-frontend
---

# Sprint 1 — Deploy Vercel (`apps/web` → globeliv.com)

> Primer deploy real del frontend a producción. **9 commits "trigger"** porque el approach correcto no era obvio; valió la pena documentar la lección.

Sprint maestro: [[Sprint 1 - Auth y Usuarios (26-27 may)]]

---

## 🎯 Lo que se quería

Que cada push a `main` despliegue automáticamente `apps/web` a Vercel sin pasos manuales y sin requerir GitHub App de Vercel (no había permisos org-level).

## ⚙️ Approach final: **Prebuilt deploy desde GitHub Actions**

Archivo: `Globeliv/.github/workflows/deploy.yml`

### Por qué prebuilt

Los Hobby accounts de Vercel que despliegan vía CLI remoto entraban a estado `BLOCKED` al detectar build de monorepo. Con prebuilt:

1. CI buildea localmente (`vercel build --prod`)
2. CI sube el artefacto firmado (`vercel deploy --prebuilt --prod`)
3. Vercel sirve el output sin re-build

Trade-off: build time se paga en GH Actions, no en infra Vercel.

### Flujo del workflow

```yaml
deploy-web:
  steps:
    - checkout
    - setup pnpm + node 22 (con cache)
    - pnpm install --frozen-lockfile
    - npm install -g vercel@latest
    - vercel pull --yes --environment=production --token=$VERCEL_TOKEN
    - vercel build --prod --token=$VERCEL_TOKEN
    - vercel deploy --prebuilt --prod --token=$VERCEL_TOKEN
```

### Concurrency

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false  # ← deploys en curso NUNCA se cancelan
```

Razón: cancelar un deploy a mitad de camino deja prod en estado inconsistente (assets parcialmente subidos). Los deploys hacen fila.

---

## 🔐 Secrets y env vars

Required en **GitHub repo secrets**:

- `VERCEL_TOKEN` — token de cuenta personal con scope deploy

Required en **Vercel project** (`vercel pull` los baja):

- `NEXT_PUBLIC_API_URL=https://api.globeliv.com`
- Variables públicas del frontend

El archivo `.vercel/project.json` está **commiteado** en el repo. Tiene `projectId` y `orgId` pero **no** el token (eso lo da el workflow).

---

## 🐞 Lecciones aprendidas (9 commits trigger)

| Síntoma | Causa | Fix |
|---|---|---|
| Vercel `BLOCKED` state al deploy desde CLI | Hobby account + monorepo + CLI remoto | Cambiar a prebuilt approach |
| Author "GitHub Actions <noreply@...>" rechazado | Vercel requiere git author verificado | Configurar `git config user.email` antes del deploy en el workflow |
| Builds canceleados unos a otros | `cancel-in-progress: true` por default | `cancel-in-progress: false` (deploys hacen fila) |
| Build "succeeded" pero `globeliv.com` 404 | DNS aún no apuntaba al deployment | Domain configurado en Vercel UI tras primer deploy verde |

---

## 🌐 Dominio

- **Vercel preview deploys:** `*.vercel.app` (incluidos en CORS regex del API)
- **Production:** `globeliv.com` (redirige 307 a `www.globeliv.com`)
- **DNS:** A + CNAME apuntan a Vercel
- **TLS:** auto-managed por Vercel (Let's Encrypt)

> El redirect `globeliv.com` → `www.globeliv.com` rompió el CORS del backend inicialmente — fix en [[Sprint 1 — CORS y Cookies cross-domain]].

---

## 🧪 Validación tras deploy verde

Pruebas manuales que se corren tras cada push a `main`:

1. Abrir `https://www.globeliv.com` → carga `/welcome` o `/signin`
2. Console del browser → sin errors de CORS
3. Network tab → `https://api.globeliv.com/trpc/...` retorna 200
4. `/profile` accesible tras signin

---

## 📂 Archivos relevantes

- `Globeliv/.github/workflows/deploy.yml` — workflow de deploy
- `Globeliv/.github/workflows/ci.yml` — lint + typecheck + build (gate antes del deploy)
- `Globeliv/.vercel/project.json` — link al proyecto Vercel
- `Globeliv/apps/web/next.config.ts` — config Next (rewrites, headers)
- `Globeliv/apps/web/.env.local` — env vars de dev (no commit)
