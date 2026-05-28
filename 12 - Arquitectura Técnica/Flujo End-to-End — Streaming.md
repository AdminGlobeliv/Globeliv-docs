---
sección: arquitectura
tema: flujo-streaming
---

# Flujo End-to-End — Streaming

> El core del producto. **El video NUNCA pasa por nuestros servers** — Agora hace todo el RTC. Nosotros solo emitimos tokens firmados y guardamos metadata.

---

## 🎬 Secuencia 1 — Go Live (streamer crea transmisión)

```mermaid
sequenceDiagram
    autonumber
    actor S as Streamer
    participant W as www.globeliv.com<br/>(/go-live)
    participant A as api.globeliv.com<br/>(/trpc/streams.create)
    participant DB as Postgres
    participant Ag as Agora cloud

    S->>W: Abre /go-live
    W->>A: auth.me (verifica login)
    A-->>W: { user }
    S->>W: Permite cámara (getUserMedia local)
    Note over W,S: Preview cámara LOCAL — no consume Agora aún
    S->>W: Llena título + monetización + location
    W->>W: useReverseGeocode (lat/lng → "Arequipa, Perú")
    S->>W: Click "Iniciar transmisión"
    W->>A: streams.create({title, monetization, location*})

    A->>A: Validación: monetization === 'free'?
    Note over A: Sprint 2: rechaza pagados<br/>(rating ≥ 4.5 y ≥ 20 streams)

    A->>A: randomUUID() = agoraChannelId
    A->>DB: INSERT streams VALUES (..., status='live')

    alt user ya tenía stream live
        DB-->>A: 23505 (partial unique violation)
        A-->>W: CONFLICT "Ya tienes una transmisión en vivo"
    else inserted OK
        DB-->>A: { streamId, agoraChannelId, startedAt }
        A->>A: tokenService.issueHostToken({channelId, userId})
        A-->>W: { streamId, agora: hostToken }
    end

    W->>W: sessionStorage[host-token:{streamId}] = hostToken
    W->>S: router.push('/watch/{streamId}?role=host')

    Note over W,Ag: En la pantalla Player:
    W->>Ag: AgoraRTC.join(appId, channelId, hostToken, uid)
    W->>Ag: createMicrophoneAudioTrack + createCameraVideoTrack
    W->>Ag: client.publish([audio, video])
    Note over W,Ag: Streamer transmite (video va a Agora, NO a nuestra API)
```

### Decisiones críticas

| Decisión | Razón |
|---|---|
| `agora_channel_id` = `randomUUID()` server-side | No leakear identificadores internos al SDK Agora |
| Partial unique `(user_id) WHERE status='live'` | DB enforcea "1 live por user" — sin race condition |
| Mock token service | Demos funcionan sin credenciales Agora — ver [[Sprint 2 — Agora Token Service (mock + real)]] |
| hostToken en sessionStorage entre /go-live y /watch | TTL 4h del token, evita viaje extra al server |

---

## 📺 Secuencia 2 — Viewer ve el stream

```mermaid
sequenceDiagram
    autonumber
    actor V as Viewer
    participant W as www.globeliv.com<br/>(/watch/[id])
    participant A as api.globeliv.com
    participant DB as Postgres
    participant Ag as Agora cloud

    V->>W: Abre /watch/{streamId}
    W->>A: streams.byId({streamId})
    A->>DB: SELECT stream JOIN users
    DB-->>A: { stream, streamer }
    A-->>W: { stream }

    alt stream.status !== 'live'
        W->>V: "Ya no está en vivo"
    else live, role=viewer
        W->>A: streams.joinAsViewer({streamId})
        A->>A: SELECT stream WHERE id = ? AND status='live'
        A->>A: userIdForUid = ctx.userId ?? `anon-${uuid()}`
        A->>A: tokenService.issueViewerToken(...)
        A-->>W: { agora: viewerToken }

        W->>Ag: AgoraRTC.join(appId, channelId, viewerToken, uid)
        Ag-->>W: event 'user-published'<br/>(streamer publicó tracks)
        W->>Ag: client.subscribe(remoteUser, 'video')
        W->>Ag: client.subscribe(remoteUser, 'audio')
        Note over W,V: Viewer ve y escucha al streamer
    end
```

### Por qué `streams.byId` separado de `streams.joinAsViewer`

- **Separación de concerns:** lectura (cacheable, no efecto colateral) vs emisión de token (mutación)
- **UX:** si el stream ya terminó (`status='ended'`), el viewer ve la pantalla "ya no está en vivo" **sin gastar un viewerToken** para nada
- **Share links:** mandar `/watch/xyz` a alguien funciona aunque el stream haya terminado — no rompe

### Por qué viewer anónimo está permitido (Sprint 2)

`joinAsViewer` es `publicProcedure` (no `protected`). Razón:
- Share-link friendly — alguien que recibe el link no tiene que registrarse para mirar
- `uid` efímero `anon-${uuid()}` por sesión — Agora lo trata como viewer único
- Si aparece abuso (bots, scraping), Sprint 4+ endurece con rate-limit por IP

---

## 🛑 Secuencia 3 — End stream (host termina)

```mermaid
sequenceDiagram
    autonumber
    actor S as Streamer
    participant W as www.globeliv.com
    participant A as api.globeliv.com<br/>(/trpc/streams.end)
    participant DB as Postgres
    participant Ag as Agora cloud

    S->>W: Click "Terminar transmisión"
    W->>Ag: client.leave() + tracks.close()
    W->>A: streams.end({streamId})
    A->>DB: UPDATE streams<br/>SET status='ended', ended_at=now()<br/>WHERE id=? AND user_id=ctx.userId<br/>AND status='live' AND deleted_at IS NULL<br/>RETURNING { id, ended_at }

    alt 1 row updated
        A-->>W: { ok: true, endedAt }
    else 0 rows (no existe / no tuyo / ya ended)
        A->>DB: SELECT para distinguir caso
        alt no existe
            A-->>W: NOT_FOUND
        else no es tuyo
            A-->>W: FORBIDDEN
        else ya ended
            A-->>W: { ok: true, endedAt: null }<br/>(idempotente — UX más suave que error)
        end
    end

    W->>S: router.push('/profile')
```

### Por qué `UPDATE … WHERE precondición RETURNING`

Operación **atómica** — la transición `live → ended` no permite doble end. Si el usuario hace doble tap al botón:

- Request 1: UPDATE pasa → `live → ended`. Devuelve `ok`.
- Request 2: UPDATE no afecta filas (precondición `status='live'` ya no se cumple) → fallback idempotente devuelve `ok`.

Mejor UX que un toast de error.

---

## 📊 Diagrama de estados de un stream

```mermaid
stateDiagram-v2
    [*] --> live: streams.create
    live --> ended: streams.end (manual)
    live --> error: Agora corta (mod automática, network)
    live --> live: viewers join/leave<br/>(viewer_count cambia)
    ended --> [*]: soft delete<br/>(o queda en histórico)
    error --> [*]: soft delete<br/>(filtrable en analytics)
```

`error` se separa de `ended` para poder filtrar "streams buenos" en analytics y replays.

---

## 🚀 Por qué este diseño aguanta tráfico

Aplicando §11.1 del CLAUDE.md ("escalabilidad como criterio único"):

| Carga | Mecanismo | Aguanta |
|---|---|---|
| Video bidireccional | Agora cloud lo maneja, no nosotros | Cientos de miles de streams concurrentes |
| Lista Home (listLive) | Index `(status, started_at desc)` | Millones de streams ended sin afectar la query |
| Emisión de tokens | Stateless, ~5ms por token | Cientos por segundo en una sola instancia |
| Viewer count realtime | Sprint 3: Redis pub/sub (hot path) | 10k+ updates por segundo |
| Chat realtime | Sprint 3: Socket.io con adaptador Redis | 1k+ msg/seg por stream |

> El cuello de botella **no es nuestro stack** — es Agora (cuando lleguemos a 10k+ streams concurrentes negociamos plan enterprise) y Redis (clustering trivial cuando aplique).

---

## 🔗 Notas relacionadas

- [[Modelo de Datos]] — schema `streams` + indexes
- [[Sprint 2 — Schema de Streams]] — diseño de la tabla con razones
- [[Sprint 2 — tRPC Streams Router]] — código de los procedures
- [[Sprint 2 — Agora Token Service (mock + real)]] — patrón mock/real
- [[Sprint 2 — Pantallas Go Live y Watch]] — frontend
- [[Pantalla 4 - Go Live]], [[Pantalla 3 - Player de Stream]] — spec de producto
