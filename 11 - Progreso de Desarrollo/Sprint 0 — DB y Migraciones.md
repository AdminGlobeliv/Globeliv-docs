---
sprint: 0
fecha: 2026-05-25
tema: base-de-datos
---

# Sprint 0 — Base de datos y migraciones

> Drizzle setup + schema inicial. Solo `users` en Sprint 0 — las demás tablas entran cuando su sprint las pida.

Sprint maestro: [[Sprint 0 - Setup (25 may)]]

---

## 🗄 Tabla `users` — estado al cerrar Sprint 0

| Columna | Tipo | Notas |
|---|---|---|
| `id` | `uuid` PK | `default gen_random_uuid()` |
| `email` | `text NOT NULL` | identidad natural |
| `password_hash` | `text NOT NULL` (luego nullable en S1) | bcrypt(12) |
| `display_name` | `text NOT NULL` | mostrado en UI |
| `username` | `text` (nullable) | onboarding lo pide después |
| `avatar_url` | `text` | data URL en S1 → R2 cuando aplique |
| `role` | enum `user_role` | `viewer` \| `streamer` \| `admin` |
| `status` | enum `user_status` | `active` \| `suspended` \| `banned` |
| `email_verified` | `boolean` | default `false` |
| `created_at` | `timestamptz NOT NULL DEFAULT now()` | — |
| `updated_at` | `timestamptz NOT NULL DEFAULT now()` | `$onUpdateFn` |
| `deleted_at` | `timestamptz` (nullable) | **soft delete obligatorio** |

**Indexes:**
- `users_email_unique_idx` — UNIQUE funcional sobre `lower(email)` (case-insensitive)
- `users_role_idx`
- `users_status_idx`

> Cambio respecto al CLAUDE.md original: la identidad inicial era `phone` (OTP por SMS). Se pivotó a **email + password** en el primer commit del schema. La razón: deferimos Twilio hasta tener tráfico real (Sprint 6+) y email es 0-cost. `phone` se reintroducirá cuando aparezca 2FA / notificaciones SMS opcionales.

Archivo schema: `Globeliv/packages/database/src/schema/users.ts`

---

## 📜 Migración inicial

**`0000_fancy_kat_farrell.sql`** — generada con `drizzle-kit generate`.

```sql
CREATE TYPE "public"."user_role" AS ENUM('viewer', 'streamer', 'admin');
CREATE TYPE "public"."user_status" AS ENUM('active', 'suspended', 'banned');

CREATE TABLE "users" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL,
  "email" text NOT NULL,
  "password_hash" text NOT NULL,
  "display_name" text NOT NULL,
  "username" text,
  "avatar_url" text,
  "role" "user_role" DEFAULT 'viewer' NOT NULL,
  "status" "user_status" DEFAULT 'active' NOT NULL,
  "email_verified" boolean DEFAULT false NOT NULL,
  "created_at" timestamp with time zone DEFAULT now() NOT NULL,
  "updated_at" timestamp with time zone DEFAULT now() NOT NULL,
  "deleted_at" timestamp with time zone
);

CREATE UNIQUE INDEX "users_email_unique_idx"
  ON "users" USING btree (lower("email"));
CREATE INDEX "users_role_idx" ON "users" USING btree ("role");
CREATE INDEX "users_status_idx" ON "users" USING btree ("status");
```

> Nombre original tracked en git: `0000_complete_khan.sql` — renombrada a `0000_fancy_kat_farrell.sql` cuando se regeneraron snapshots tras la decisión de email-first.

---

## 🛠 Drizzle — configuración

- ORM: `drizzle-orm` `^0.38.x`
- CLI: `drizzle-kit`
- Driver: `postgres` (postgres.js)
- Config: `Globeliv/packages/database/drizzle.config.ts`

**Scripts (desde el root del repo):**

```bash
# Generar migración desde schema modificado
pnpm --filter @globeliv/database db:generate

# Aplicar migraciones al DB conectado vía DATABASE_URL
pnpm --filter @globeliv/database db:migrate

# GUI Drizzle Studio
pnpm --filter @globeliv/database db:studio
```

---

## 🧬 Patrón de schema

Reglas que se aplican a TODA tabla nueva (vienen de `CLAUDE.md` y `ARCHITECTURE_PROMPT.md`):

1. **`id` UUID primary key** con `gen_random_uuid()` — no autoincrement.
2. **Triple timestamp obligatorio:** `created_at`, `updated_at`, `deleted_at` (todas `timestamptz`).
3. **Soft delete por default** — `deleted_at IS NULL` en TODO filtro de lectura. NUNCA `DELETE` físico salvo GDPR.
4. **`updated_at` con `$onUpdateFn(() => new Date())`** — Drizzle lo setea en cada update sin trigger SQL.
5. **Indexes en el mismo PR que la query** — si una feature lee por `(status, started_at)`, ese index entra junto.
6. **Multi-tenant:** TODA query a tabla con `user_id` lleva `WHERE user_id = ${ctx.userId}`. Sin excepciones.

---

## 📂 Layout de `packages/database/`

```
packages/database/
├── drizzle.config.ts
├── migrations/
│   ├── 0000_fancy_kat_farrell.sql
│   └── meta/
│       ├── _journal.json
│       └── 0000_snapshot.json
└── src/
    ├── index.ts          ← re-exports schema, db client, helpers
    ├── client.ts         ← postgres.js + drizzle instance
    └── schema/
        ├── index.ts
        ├── enums.ts
        ├── users.ts
        └── relations.ts
```

---

## 🔗 Lo que sigue

Migraciones que entran en sprints posteriores (registradas en [[Sprint 1 — Sistema de Auth]] y [[Sprint 2 — Schema de Streams]]):

- `0001_flat_zarda.sql` (S1, 26 may) — agrega `location`, unique index `username`
- `0002_amazing_whistler.sql` (S1, 26 may) — `password_hash` nullable, `google_id` + index
- `0003_productive_ken_ellis.sql` (S2, 27 may) — tabla `streams` + enums + indexes
