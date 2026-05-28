---
sprint: 2
fecha: 2026-05-27
tema: schema
---

# Sprint 2 — Schema de Streams

> Tabla `streams` + 2 enums + 4 indexes. Diseñada para responder al Home Explorar barato (index compuesto status+started_at) y bloquear la regla "1 stream live por user" a nivel DB.

Sprint maestro: [[Sprint 2 - Streaming Core (27-28 may)]]

---

## 🗄 Tabla `streams`

**Archivo:** `Globeliv/packages/database/src/schema/streams.ts`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | `uuid` PK | `default gen_random_uuid()` |
| `user_id` | `uuid NOT NULL` | FK a `users.id`, `ON DELETE CASCADE` |
| `title` | `text NOT NULL` | mostrado en Home + Player |
| `status` | enum `stream_status` | `live` (default) \| `ended` \| `error` |
| `monetization` | enum `monetization_level` | `free` (default) \| `access_1` \| `access_3` \| `access_5` |
| `agora_channel_id` | `text NOT NULL` | UUID generado server-side, único |
| `viewer_count` | `integer NOT NULL DEFAULT 0` | snapshot — hot-path va a Redis en S3 |
| `peak_viewer_count` | `integer NOT NULL DEFAULT 0` | máximo histórico del stream |
| `location_text` | `text` | "Arequipa, Perú" — mostrado al viewer |
| `location_lat` | `numeric(9, 6)` | precision ~10cm, sobrada |
| `location_lng` | `numeric(9, 6)` | usada en filtro "Cerca" (S3+) y Misiones (S5) |
| `thumbnail_url` | `text` | placeholder hasta S6 (Agora REST captura) |
| `started_at` | `timestamptz NOT NULL DEFAULT now()` | — |
| `ended_at` | `timestamptz` | NULL hasta que `status` cambie |
| `created_at` | `timestamptz NOT NULL` | — |
| `updated_at` | `timestamptz NOT NULL` | `$onUpdateFn` |
| `deleted_at` | `timestamptz` | soft delete |

## 🔣 Enums

**Archivo:** `Globeliv/packages/database/src/schema/enums.ts`

```ts
export const streamStatusEnum = pgEnum("stream_status", ["live", "ended", "error"]);
//                                                                    ^^^^^^^^
// `error` cubre Agora cortando la sesión sin que el host haga end manual
// (ej. moderación automática termina el stream). Lo separamos de `ended`
// para poder filtrar streams "buenos" en analytics y replays.

export const monetizationLevelEnum = pgEnum("monetization_level", [
  "free",
  "access_1",  // $1 access
  "access_3",  // $3 access
  "access_5",  // $5 access
]);
// Regla UX (Pantalla 4): rating >= 4.5 y >= 20 streams para desbloquear
// los pagados. Enforce en router tRPC, no DB — admin puede setear manual.
```

---

## 🔑 Indexes

```ts
[
  // Channel ID único — si Agora recicla nombres terminamos con dos streams
  // apuntando al mismo canal y el viewer se uniría al equivocado.
  uniqueIndex("streams_channel_unique_idx").on(table.agoraChannelId),

  // Home: lista streams `live` ordenados por más recientes primero.
  index("streams_status_started_idx")
    .on(table.status, table.startedAt.desc()),

  // Perfil de streamer: historial ordenado.
  index("streams_user_started_idx")
    .on(table.userId, table.startedAt.desc()),

  // Regla de negocio: un usuario solo puede tener un stream `live` a la vez.
  // Partial unique para que streams `ended`/`error` no bloqueen nuevos.
  // Excluye soft-deleted.
  uniqueIndex("streams_one_live_per_user_idx")
    .on(table.userId)
    .where(sql`${table.status} = 'live' AND ${table.deletedAt} IS NULL`),
]
```

### Por qué el partial unique

Sin él, "un live por user" se valida en aplicación: `SELECT count + INSERT`. Esto **es racy** — dos requests simultáneas pueden ambas leer 0 e insertar dos. Con partial unique, **Postgres rechaza el segundo INSERT** con `23505` (unique violation). El router lo mapea a `TRPCError({ code: 'CONFLICT' })`.

Ver implementación en [[Sprint 2 — tRPC Streams Router]] (`streams.create`).

---

## 📜 Migración

**`0003_productive_ken_ellis.sql`** (2026-05-27 16:30):

```sql
CREATE TYPE "public"."monetization_level" AS ENUM('free','access_1','access_3','access_5');
CREATE TYPE "public"."stream_status" AS ENUM('live','ended','error');

CREATE TABLE "streams" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL,
  "user_id" uuid NOT NULL,
  "title" text NOT NULL,
  "status" "stream_status" DEFAULT 'live' NOT NULL,
  "monetization" "monetization_level" DEFAULT 'free' NOT NULL,
  "agora_channel_id" text NOT NULL,
  "viewer_count" integer DEFAULT 0 NOT NULL,
  "peak_viewer_count" integer DEFAULT 0 NOT NULL,
  "location_text" text,
  "location_lat" numeric(9, 6),
  "location_lng" numeric(9, 6),
  "thumbnail_url" text,
  "started_at" timestamp with time zone DEFAULT now() NOT NULL,
  "ended_at" timestamp with time zone,
  "created_at" timestamp with time zone DEFAULT now() NOT NULL,
  "updated_at" timestamp with time zone DEFAULT now() NOT NULL,
  "deleted_at" timestamp with time zone
);

ALTER TABLE "streams" ADD CONSTRAINT "streams_user_id_users_id_fk"
  FOREIGN KEY ("user_id") REFERENCES "public"."users"("id")
  ON DELETE cascade ON UPDATE no action;

CREATE UNIQUE INDEX "streams_channel_unique_idx"
  ON "streams" USING btree ("agora_channel_id");

CREATE INDEX "streams_status_started_idx"
  ON "streams" USING btree ("status","started_at" DESC NULLS LAST);

CREATE INDEX "streams_user_started_idx"
  ON "streams" USING btree ("user_id","started_at" DESC NULLS LAST);

CREATE UNIQUE INDEX "streams_one_live_per_user_idx"
  ON "streams" USING btree ("user_id")
  WHERE "streams"."status" = 'live' AND "streams"."deleted_at" IS NULL;
```

Aplicada con:

```bash
pnpm --filter @globeliv/database db:migrate
```

---

## 🚀 Por qué este diseño aguanta 10x el tráfico

Aplicando la regla §11.1 del `CLAUDE.md` ("escalabilidad como criterio único"):

| Query | Index que la cubre | Razonamiento |
|---|---|---|
| `listLive` (Home) — `WHERE status='live' ORDER BY started_at DESC LIMIT 24` | `streams_status_started_idx` | Compound (status, started_at desc) — Postgres devuelve sin sort extra |
| `byId` (Player) | PK | trivial |
| `joinAsViewer` — `WHERE id = ?` | PK | trivial |
| `create` — chequeo "1 live" | `streams_one_live_per_user_idx` (partial) | Postgres lo enforcea sin SELECT previo |
| `end` — `WHERE id=? AND user_id=? AND status='live'` | PK + `streams_user_started_idx` | PK basta, idx ayuda perfiles |

**Lectura "más reciente" (Home) NO escanea toda la tabla**, ni siquiera con millones de streams `ended`: el index parcial-ish `(status, started_at desc)` se materializa por status y el orden lo da el btree directamente.

---

## 🔗 Próximas tablas (sprints siguientes)

Estas todavía no existen — se documentarán en su propio sprint:

- `chat_messages`, `follows` → Sprint 3
- `tips`, `payouts`, `wallets` → Sprint 4
- `requests`, `escrow` → Sprint 5
- `passport_*`, `notifications` → Sprint 6
- `reports`, `moderation_actions` → Sprint 7
