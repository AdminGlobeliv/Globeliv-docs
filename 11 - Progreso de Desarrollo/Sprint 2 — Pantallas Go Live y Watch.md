---
sprint: 2
fecha: 2026-05-27
tema: frontend
---

# Sprint 2 — Pantallas Go Live y Watch

> Implementación de la [[Pantalla 4 - Go Live]] (crear transmisión) y la [[Pantalla 3 - Player de Stream]] (verla). Cliente Agora dual (mock + real) sigue el mismo patrón del backend.

Sprint maestro: [[Sprint 2 - Streaming Core (27-29 may)]]

---

## 📱 `/go-live` — Pantalla 4

**Archivos:**
- `Globeliv/apps/web/src/app/go-live/page.tsx` — page con guard de auth
- `Globeliv/apps/web/src/app/go-live/_components/go-live-form.tsx` — form principal
- `Globeliv/apps/web/src/app/go-live/_components/camera-preview.tsx` — preview cámara local

### Flow

```
1. useEffect: auth.me.useQuery — si UNAUTHORIZED → redirect /signin?next=/go-live
2. Form con campos:
   - Título (required, max 80 chars)
   - Monetización (radio): free / access_1 / access_3 / access_5 (3 últimos con candado)
   - Location: detect → reverse-geocode → texto editable
3. Preview de cámara LOCAL (getUserMedia) — no consume créditos Agora
4. Click "Iniciar transmisión":
   - trpc.streams.create.mutate({ title, monetization, locationText, locationLat, locationLng })
   - onSuccess: guarda hostToken en sessionStorage['globeliv:host-token:${streamId}']
   - router.push(`/watch/${streamId}?role=host`)
   - onError: mapea CONFLICT / PRECONDITION_FAILED / UNAUTHORIZED a mensajes de UI
```

### Reverse geocoding

**Archivo:** `Globeliv/apps/web/src/lib/geo/use-reverse-geocode.ts`

- Pide permiso de geo al usuario
- Llama a un endpoint reverse-geocode (provider depende de env — `NEXT_PUBLIC_GEOCODE_PROVIDER`)
- Devuelve texto editable ("Arequipa, Perú") que el usuario puede ajustar
- Las coords `lat`/`lng` viajan al backend pero no se muestran (privacidad por default)

### Decisión: hostToken en sessionStorage

Razones para no pedir el token de nuevo en `/watch`:

- El token de `streams.create` ya está firmado para este channel
- TTL host 4h → suficiente para una transmisión típica
- Evita un viaje extra al servidor

Razones para usar **sessionStorage** y no localStorage:

- Limpia al cerrar tab → no quedan tokens huérfanos
- Aislado per-tab → no leak entre sesiones

Manejo de error: si `sessionStorage` está bloqueado (modo privado en algunos browsers), el Player detecta la ausencia y muestra "No se pudo recuperar tu token. Recarga la página."

---

## 📺 `/watch/[id]` — Pantalla 3

**Archivos:**
- `Globeliv/apps/web/src/app/watch/[id]/page.tsx` — page con role detection
- `Globeliv/apps/web/src/app/watch/[id]/_components/player-video.tsx` — `<video>` + Agora attach
- `Globeliv/apps/web/src/app/watch/[id]/_components/player-actions-bar.tsx` — controles (mute, fullscreen, end)
- `Globeliv/apps/web/src/app/watch/[id]/_components/chat-placeholder.tsx` — chat dummy hasta S3
- `Globeliv/apps/web/src/app/watch/[id]/_components/tip-quick-buttons.tsx` — tips dummy hasta S4

### Dos modos según `?role=host`

| | host | viewer |
|---|---|---|
| Token source | `sessionStorage` (set por /go-live) | `streams.joinAsViewer.mutate` |
| Tracks | publica cámara + mic | subscribe a tracks remotos |
| Acciones | "Terminar transmisión" | "Salir" |

### Layout responsive

- **Mobile / tablet:** video full-width, debajo en stack: actions → tips → chat
- **Desktop ≥lg:** video (60% ancho) a la izquierda, sidebar derecho (40%) con chat
- Coincide con prototipo del spec en formato wide

### Pre-fetch separado de token

El `/watch` hace dos pasos:

1. `streams.byId.useQuery({ streamId })` — metadata (cacheable, no gasta token)
2. Solo si modo viewer y stream `live`: `streams.joinAsViewer.useMutation()` para token

Razón: si llegas a `/watch/[id]` de un stream **ya terminado** (share link expirado), ves metadata + mensaje "Ya no está en vivo" sin gastar un viewer token para nada.

---

## 🎬 Cliente Agora dual

Mismo patrón que el backend: interfaz + dos implementaciones.

**Archivos:**
- `Globeliv/apps/web/src/lib/agora/types.ts` — `StreamClient` interface
- `Globeliv/apps/web/src/lib/agora/real-client.ts` — wrapper de `agora-rtc-sdk-ng`
- `Globeliv/apps/web/src/lib/agora/mock-client.ts` — no-op con eventos sintéticos
- `Globeliv/apps/web/src/lib/agora/use-stream-client.ts` — React hook
- `Globeliv/apps/web/src/lib/agora/index.ts` — `getAgoraAppId()`, swap por env

### Swap

```ts
// apps/web/src/lib/agora/index.ts
export function createStreamClient(): StreamClient {
  if (process.env.NEXT_PUBLIC_AGORA_USE_MOCK === "true") {
    return new MockStreamClient();
  }
  return new RealStreamClient();
}
```

Hook React `useStreamClient(token, role, channelId)` encapsula:
- `useEffect` para init/join al montar
- Cleanup al desmontar (leave + close tracks)
- Estado `connected | connecting | error`
- Eventos `user-published`, `user-unpublished` mapeados a state

---

## 🏠 Home Explorar (`apps/web/src/app/page.tsx`)

**Archivos:**
- `Globeliv/apps/web/src/app/page.tsx` — Home
- `Globeliv/apps/web/src/app/_components/filters.tsx` — chips de filtro
- `Globeliv/apps/web/src/app/_components/stream-card.tsx` — card de stream

### Implementación

```tsx
const { data, isLoading } = trpc.streams.listLive.useQuery({ limit: 24 });

return (
  <main>
    <TopNav />
    <Filters />  {/* placeholders: Cerca, Trending, Live ahora */}
    <Grid>
      {data?.streams.map(s => <StreamCard key={s.id} stream={s} />)}
    </Grid>
    <BottomNav />
  </main>
);
```

### StreamCard

- Thumbnail (placeholder gradient hasta S6)
- Badge `LIVE` rojo con dot animado (token: `--color-brand-red`)
- Title + streamer avatar + display name
- `locationText` con ícono pin
- `viewerCount` con ícono ojo

Click → `router.push(`/watch/${stream.id}`)`

### Filters

Por ahora son **placeholders visuales**. Sprint 3 los conectará con:
- "Cerca" → ordenar por `location` proximity
- "Trending" → por `viewer_count` desc
- "Live ahora" → ya es el default

---

## 🎨 Componentes UI nuevos del sprint

Ubicados en `Globeliv/apps/web/src/app/_components/` (locales a Home) y `src/app/watch/[id]/_components/` (locales a Watch).

| Componente | Función |
|---|---|
| `StreamCard` | Card en Home con thumbnail + meta |
| `Filters` | Chips de filtro (placeholder funcional) |
| `PlayerVideo` | Wrapper `<video>` + Agora attach |
| `PlayerActionsBar` | Bottom bar con mute/fullscreen/end (responsive) |
| `ChatPlaceholder` | Mensajes mock, input deshabilitado con badge "Pronto" |
| `TipQuickButtons` | Botones $1/$3/$5 deshabilitados con badge "Sprint 4" |

> Todos respetan [[heroui_first]]: cuando HeroUI tiene el componente (Button, Card, Spinner) se usa vía `@globeliv/ui`. Custom solo donde HeroUI no llega (video element con overlays, chat layout).

---

## 🧪 Validación end-to-end con mock

Con `AGORA_USE_MOCK=true` en api y `NEXT_PUBLIC_AGORA_USE_MOCK=true` en web:

1. Login → `/go-live` → form muestra preview cámara local (getUserMedia real, no Agora)
2. Submit → toast OK → redirect `/watch/[streamId]?role=host`
3. Player muestra "Conectado al canal (mock)" con preview cámara
4. En otra ventana incógnito: `/` muestra el stream → click → `/watch/[streamId]`
5. Viewer ve "Conectado al canal (mock)" con video placeholder
6. Host click "Terminar" → `streams.end` → ambos redirigen o ven "Ya no está en vivo"

Lo que NO se valida con mock: video real, audio real, latencia de red Agora.

---

## 🔗 Referencias

- Pantalla original: [[Pantalla 4 - Go Live]], [[Pantalla 3 - Player de Stream]]
- Backend: [[Sprint 2 — tRPC Streams Router]], [[Sprint 2 — Agora Token Service (mock + real)]]
- Reglas UX que se siguieron: [[Reglas de UX que no se negocian]]
