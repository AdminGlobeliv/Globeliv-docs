---
sprint: 2
fecha: 2026-05-27
tema: trpc-router
---

# Sprint 2 — tRPC Streams Router

> 5 procedures que cubren el lifecycle completo de un stream: crear, terminar, listar, detalle, y unirse como viewer.

Sprint maestro: [[Sprint 2 - Streaming Core (27-28 may)]]

---

## 📜 Procedures

**Archivo:** `Globeliv/apps/api/src/trpc/routers/streams.router.ts`

| Procedure | Tipo | Auth | Función |
|---|---|---|---|
| `streams.create` | mutation | **protected** | Crea stream `live` + emite hostToken Agora |
| `streams.end` | mutation | **protected** | Marca stream `ended` (solo owner). Idempotente. |
| `streams.listLive` | query | public | Home Explorar — lista 24 más recientes |
| `streams.byId` | query | public | Detalle para el Player (sin token) |
| `streams.joinAsViewer` | mutation | public | Emite viewerToken Agora para stream live |

---

## 🛠 `streams.create` — el más interesante

### Flow

```
1. Validar input con createStreamInput (Zod)
2. Si monetization != "free" → throw PRECONDITION_FAILED
   (regla: rating >= 4.5 y >= 20 streams; aplica desde Sprint 3+)
3. Generar agoraChannelId = randomUUID()
4. INSERT streams (...) try/catch
5. Si PG error.code === '23505' (unique violation por partial index):
   → throw CONFLICT "Ya tienes una transmisión en vivo"
6. Emit hostToken via ctx.tokenService.issueHostToken({...})
7. Return { streamId, startedAt, agora: hostToken }
```

### Por qué el insert atómico (sin SELECT previo)

El partial unique index `streams_one_live_per_user_idx` enforcea "1 live por user" a nivel DB. Si dos requests simultáneas llegan, **una INSERT pasa, la otra falla**. Sin esto sería race: `SELECT count + INSERT` permite que ambos lean 0.

```ts
try {
  await ctx.db.insert(schema.streams).values({...}).returning({...});
} catch (err) {
  if (err.code === '23505') {
    throw new TRPCError({
      code: 'CONFLICT',
      message: 'Ya tienes una transmisión en vivo. Termínala antes de iniciar otra.',
    });
  }
  throw err;
}
```

---

## 🛑 `streams.end` — idempotente

```
1. UPDATE streams SET status='ended', ended_at=now()
   WHERE id=? AND user_id=? AND status='live' AND deleted_at IS NULL
   RETURNING id, ended_at
2. Si updated.length === 0:
   - SELECT para distinguir entre:
     a. No existe → NOT_FOUND
     b. Existe pero no es tuyo → FORBIDDEN
     c. Ya estaba ended → return { ok: true, endedAt: null } (idempotente)
3. Si updated.length > 0 → return endedAt
```

### Por qué `UPDATE … WHERE precondición RETURNING`

Atomicidad — el "transition" de `live → ended` no permite doble end. Si el usuario hace doble tap al botón:

- Request 1: UPDATE pasa → `live → ended`. Devuelve `ok`.
- Request 2: UPDATE no afecta filas (precondición `status='live'` ya no se cumple) → fallback idempotente devuelve `ok`.

Mejor UX que un toast de error cuando el usuario hace doble tap.

---

## 🌐 `streams.listLive` — Home Explorar

```ts
listLive: publicProcedure
  .input(z.object({ limit: z.number().int().min(1).max(50).default(24) }).optional())
  .query(async ({ input, ctx }): Promise<{ streams: StreamListItem[] }> => {
    const limit = input?.limit ?? 24;
    const rows = await ctx.db
      .select({ ... }) // join users
      .from(schema.streams)
      .innerJoin(schema.users, eq(schema.users.id, schema.streams.userId))
      .where(and(
        eq(schema.streams.status, 'live'),
        isNull(schema.streams.deletedAt),
      ))
      .orderBy(desc(schema.streams.startedAt))
      .limit(limit);
    return { streams: rows.map(toListItem) };
  }),
```

**Performance:** usa el index `streams_status_started_idx` (status, started_at desc) → 0 sort, 0 full scan.

**Sin paginación todavía** — Sprint 2 vive con ≤200 streams concurrentes; cursor pagination llega en Sprint 3.

---

## 🔍 `streams.byId` — detalle sin token

Lectura separada de la emisión de token. Beneficios:

- El frontend puede prefetch metadata sin gastar privilegios Agora
- Cache-friendly: TanStack Query cachea por `streamId`
- Comparte el StreamPublicView con el perfil público y posibles features futuras (replays, etc.)

Devuelve **todo lo necesario para el Player** + datos del streamer (innerJoin con users).

---

## 🎟 `streams.joinAsViewer` — emisión de token

```
1. SELECT stream WHERE id=? AND deleted_at IS NULL
2. Si no existe → NOT_FOUND
3. Si status != 'live' → PRECONDITION_FAILED "Este stream ya no está en vivo"
4. userIdForUid = ctx.userId ?? `anon-${randomUUID()}`
5. token = ctx.tokenService.issueViewerToken({channelId, userId: userIdForUid})
6. Return { agora: token }
```

**Anónimos permitidos en Sprint 2.** Share-link friendly: un viewer logueado tiene uid estable; uno anónimo tiene uid efímero. Sprint 4 (pagos) endurecerá esto si aparece abuso.

---

## 🔐 `ctx.tokenService` — inyección de TokenService

El `createTrpcContext` (en `Globeliv/apps/api/src/trpc/context.ts`) recibe el singleton del TokenService:

```ts
createTrpcContext({ req, res, db, tokenService })
```

El `tokenService` se setea en `main.ts` (bootstrap) llamando a `getTokenServiceInstance()`. Detalle en [[Sprint 2 — Agora Token Service (mock + real)]].

---

## 📦 Schemas Zod compartidos

**Archivo:** `Globeliv/packages/zod-schemas/src/streams.ts`

- `createStreamInput` — `{ title, monetization, locationText?, locationLat?, locationLng? }`
- `joinStreamInput` — `{ streamId: uuid }`
- `StreamListItem` (type) — shape devuelto por `listLive`
- `StreamPublicView` (type) — shape devuelto por `byId`
- `AgoraTokenResponse` (type) — shape devuelto por host/viewer tokens

El frontend importa estos types vía `import type { ... } from "@globeliv/zod-schemas"` → tipo end-to-end sin duplicación.

---

## 🧪 Cómo probar local

Con el dev server corriendo (`pnpm --filter @globeliv/api dev`):

```bash
# 1. Crear stream (necesita cookie de auth — sacar de browser)
curl -X POST https://api.globeliv.com/trpc/streams.create \
  -H "Content-Type: application/json" \
  -H "Cookie: globeliv_access=..." \
  -d '{"title":"Test","monetization":"free","locationText":"Lima"}'

# 2. Listar
curl https://api.globeliv.com/trpc/streams.listLive

# 3. End
curl -X POST https://api.globeliv.com/trpc/streams.end \
  -H "Content-Type: application/json" \
  -H "Cookie: globeliv_access=..." \
  -d '{"streamId":"..."}'
```

---

## 🔗 Archivos relacionados

- `Globeliv/apps/api/src/trpc/routers/streams.router.ts`
- `Globeliv/apps/api/src/trpc/app-router.ts` — `streams: streamsRouter`
- `Globeliv/apps/api/src/trpc/context.ts` — inyecta `tokenService`
- `Globeliv/packages/zod-schemas/src/streams.ts`
- `Globeliv/packages/database/src/schema/streams.ts`
