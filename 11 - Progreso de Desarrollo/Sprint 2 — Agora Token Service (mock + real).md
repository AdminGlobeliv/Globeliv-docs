---
sprint: 2
fecha: 2026-05-27
tema: agora
---

# Sprint 2 — Agora Token Service (mock + real)

> Patrón **interface + dos implementaciones**. Swap atómico via env `AGORA_USE_MOCK`. Permite todo el desarrollo + CI sin credenciales reales.

Sprint maestro: [[Sprint 2 - Streaming Core (27-29 may)]]

---

## 🎯 Por qué este diseño

GlobeLiv depende de Agora para el RTC. Pero:

- En **CI** no queremos exponer credenciales productivas
- En **dev local** el equipo debería poder iterar UI sin gastar minutos Agora
- Las **demos a stakeholders** funcionan con mock — no necesitan ver video real para validar UX

**Solución:** interfaz `TokenService` + dos impls + singleton que decide al boot.

```
┌──────────────────────────────────────┐
│   getTokenServiceInstance() : TokenService
│   ├── if AGORA_USE_MOCK=true  → MockAgoraTokenService
│   └── if AGORA_USE_MOCK=false → RealAgoraTokenService
└──────────────────────────────────────┘
        │
        ▼ usado por
   ctx.tokenService.issueHostToken({...})
   ctx.tokenService.issueViewerToken({...})
```

---

## 📐 Interface

**Archivo:** `Globeliv/apps/api/src/modules/streams/token-service.interface.ts`

```ts
export interface TokenService {
  issueHostToken(args: {
    channelId: string;
    userId: string;
  }): Promise<AgoraTokenResponse>;

  issueViewerToken(args: {
    channelId: string;
    userId: string;
  }): Promise<AgoraTokenResponse>;
}

export const TOKEN_SERVICE = Symbol("TokenService");  // ← Nest DI token
```

- `channelId` = `agora_channel_id` de la fila `streams` (UUID server-side)
- `userId` = id del usuario autenticado, convertido a uint32 vía `userIdToUid()` (Agora exige numérico)

---

## ✅ `RealAgoraTokenService`

**Archivo:** `Globeliv/apps/api/src/modules/streams/real-agora-token.service.ts`

Firma con `agora-token` (SDK oficial). Diferencias `host` vs `viewer`:

| | host | viewer |
|---|---|---|
| Role | `RtcRole.PUBLISHER` | `RtcRole.SUBSCRIBER` |
| TTL | `AGORA_HOST_TOKEN_TTL_SECONDS` (default 4h) | `AGORA_VIEWER_TOKEN_TTL_SECONDS` (default 1h) |

### Quirk: agora-token es CJS-only

```ts
// agora-token asigna a `module.exports` directo, así que con ESM hay que
// importarlo como default y desestructurar los símbolos.
import agoraToken from "agora-token";
const { RtcRole, RtcTokenBuilder } = agoraToken;
```

---

## 🎭 `MockAgoraTokenService`

**Archivo:** `Globeliv/apps/api/src/modules/streams/mock-agora-token.service.ts`

Devuelve un token con shape válido pero **sin firma real**:

```ts
const token = `mock:${role}:${channelId}:${uid}:${expiresAt}`;
return { appId, channelId, token, uid, expiresAt, role };
```

El prefix `mock:` lo hace **inconfundible** en logs — si alguna vez aparece en producción, se identifica al instante.

### Para qué sirve

- CI: corre tests sin AGORA_APP_ID/CERTIFICATE
- Dev: equipo itera UI con `AGORA_USE_MOCK=true`
- Demo: stakeholders ven la UI funcionando sin video real

El cliente Agora **también es dual** (mock + real). Cuando `NEXT_PUBLIC_AGORA_USE_MOCK=true`, el cliente mock acepta el token mock sin tocar el SDK real — el flow E2E funciona, simplemente sin video.

---

## 🧷 Singleton — `getTokenServiceInstance()`

**Archivo:** `Globeliv/apps/api/src/modules/streams/instance.ts`

```ts
let cached: TokenService | undefined;

export function getTokenServiceInstance(): TokenService {
  if (cached) return cached;
  const env = loadEnv();
  const logger = new Logger("TokenService");
  if (env.AGORA_USE_MOCK) {
    logger.warn("AGORA_USE_MOCK=true — emitiendo tokens fake");
    cached = new MockAgoraTokenService();
  } else {
    logger.log("AGORA_USE_MOCK=false — firmando tokens reales");
    cached = new RealAgoraTokenService();
  }
  return cached;
}
```

**Por qué singleton:**
- El `bootstrap()` en `main.ts` lo instancia antes de montar el tRPC middleware
- El `ctx.tokenService` lo lee de aquí
- El módulo Nest (provider) **también** lo lee de aquí — un solo binding, dos consumidores

Sin singleton tendríamos **dos instancias** (una por DI, otra en tRPC), y si una env var cambiara entre ambos `loadEnv()` quedaríamos con Mock+Real corriendo a la vez.

Mismo patrón que `getDatabaseInstance()` documentado en [[Sprint 1 — Sistema de Auth]].

---

## 🔢 `userIdToUid()` — UUID → uint32

**Archivo:** `Globeliv/apps/api/src/modules/streams/uid.ts`

Agora exige uid numérico (uint32). Nuestros user IDs son UUID. Hash determinístico estable:

```
uid = fnv1a(uuid) mod (2^31 - 1) + 1
```

(Excluimos 0 porque Agora lo trata especial.)

**Determinismo importante:** el mismo user reconectándose obtiene el mismo uid → Agora no lo trata como un viewer nuevo.

---

## 📦 Módulo Nest

**Archivo:** `Globeliv/apps/api/src/modules/streams/streams.module.ts`

Provee `TOKEN_SERVICE` vía factory:

```ts
{
  provide: TOKEN_SERVICE,
  useFactory: () => getTokenServiceInstance(),
}
```

Cualquier controller/service Nest que necesite el TokenService lo inyecta:

```ts
@Inject(TOKEN_SERVICE) private readonly tokenService: TokenService
```

> En Sprint 2 ningún controller HTTP necesita el TokenService — todo va por tRPC. Pero el provider queda registrado para Sprint 6+ cuando aparezca endpoint REST para Agora REST API callbacks.

---

## 🧪 Validación pendiente

- [ ] **Test unitario `RealAgoraTokenService`** con vectores conocidos publicados por Agora — tarea **S2-03.8** del plan original
- [ ] Verificar en producción que `RtcTokenBuilder.buildTokenWithUid` da output que el SDK Web acepta (no probado end-to-end en Railway todavía)

---

## ⚙️ Env vars relacionadas

Listadas en `Globeliv/apps/api/src/env.ts` y `.env.example`:

| Variable | Tipo | Default | Uso |
|---|---|---|---|
| `AGORA_USE_MOCK` | bool | `false` | Si `true` → MockTokenService |
| `AGORA_APP_ID` | string | — | Required cuando USE_MOCK=false |
| `AGORA_APP_CERTIFICATE` | string | — | Required cuando USE_MOCK=false |
| `AGORA_HOST_TOKEN_TTL_SECONDS` | number | `14400` (4h) | TTL del host token |
| `AGORA_VIEWER_TOKEN_TTL_SECONDS` | number | `3600` (1h) | TTL del viewer token |

La validación de `APP_ID + CERTIFICATE` requeridos cuando `USE_MOCK=false` está en `env.ts` con Zod refinement → la app no arranca si la config es inconsistente.

---

## 🔗 Archivos del módulo

```
apps/api/src/modules/streams/
├── token-service.interface.ts   ← contrato
├── mock-agora-token.service.ts
├── real-agora-token.service.ts
├── instance.ts                  ← singleton factory
├── uid.ts                       ← UUID → uint32
└── streams.module.ts            ← provider Nest
```
