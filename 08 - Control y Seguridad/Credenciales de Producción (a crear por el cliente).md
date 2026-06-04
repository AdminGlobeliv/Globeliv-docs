# Credenciales de Producción — a crear por el CLIENTE

> Estas 3 integraciones externas hoy corren en **modo mock** (la app funciona,
> pero sin enviar push real / sin grabar replays / con webcams de muestra).
> Para activarlas en producción se necesitan **cuentas del cliente** — no del
> desarrollador — porque implican **facturación, propiedad de datos y
> continuidad** del producto.
>
> El cliente crea las cuentas, extrae los valores indicados y los envía. El
> equipo de desarrollo los configura (variables de entorno) y voltea el flag
> `*_USE_MOCK=false`. **No hay cambios de código.**
>
> Importante: el flag solo se pone en `false` cuando TODAS las variables de esa
> integración están cargadas (el arranque valida esto y falla si falta alguna).

---

## 1. Windy — Webcams de respaldo (lo más simple, gratis)

**Crear:** cuenta en https://api.windy.com → sección **Webcams API** → generar una **API key** (plan gratuito disponible).

**Enviar:**
| Valor | Variable | Dónde se configura |
|---|---|---|
| API key de Webcams | `WINDY_WEBCAMS_API_KEY` | Railway → servicio **API (Globeliv)** |

Activación: `WINDY_WEBCAMS_USE_MOCK=false` en el servicio API.

**Costo:** gratis (con límites de uso del plan free).

---

## 2. Firebase Cloud Messaging — Notificaciones push reales

**Crear:** un proyecto en https://console.firebase.google.com (Cloud Messaging
queda habilitado por defecto).

### a) Backend (service account)
Configuración del proyecto → **Cuentas de servicio** → *Generar nueva clave
privada* → descarga un JSON. De ese JSON saca:

| Campo del JSON | Variable | Dónde |
|---|---|---|
| `project_id` | `FCM_PROJECT_ID` | Railway → servicio **Workers (Globeliv-workers)** |
| `client_email` | `FCM_CLIENT_EMAIL` | Railway → **Workers** |
| `private_key` | `FCM_PRIVATE_KEY` | Railway → **Workers** |

Activación: `NOTIFICATIONS_USE_MOCK=false` en el servicio **Workers**.

### b) Frontend (web push)
Configuración del proyecto → **Tus apps** → agregar app **Web** → copia el
objeto de configuración. Y en **Cloud Messaging → Web Push certificates** →
*Generar par de claves* → copia la **clave pública (VAPID)**.

| Valor | Variable | Dónde |
|---|---|---|
| `apiKey` | `NEXT_PUBLIC_FIREBASE_API_KEY` | Vercel → **globeliv-web** |
| `authDomain` | `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | Vercel |
| `projectId` | `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | Vercel |
| `messagingSenderId` | `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | Vercel |
| `appId` | `NEXT_PUBLIC_FIREBASE_APP_ID` | Vercel |
| Clave pública VAPID | `NEXT_PUBLIC_FIREBASE_VAPID_KEY` | Vercel |

**Costo:** gratis (FCM no tiene costo por mensajes).

---

## 3. Agora Cloud Recording + Cloudflare R2 — Replays reales

Esta es la más laboriosa: son **dos** servicios (Agora graba y sube los
archivos directo al bucket R2). Usar el **mismo proyecto Agora** que ya emite
los tokens de streaming.

### a) Agora — Cloud Recording (REST)
En https://console.agora.io → tu proyecto → habilitar **Cloud Recording** →
sección **RESTful API** → generar un par **Customer ID / Customer Secret**.

| Valor | Variable | Dónde |
|---|---|---|
| Customer ID | `AGORA_RECORDING_CUSTOMER_ID` | Railway → servicio **API (Globeliv)** |
| Customer Secret | `AGORA_RECORDING_CUSTOMER_SECRET` | Railway → **API** |

### b) Cloudflare R2 — almacenamiento de los videos
En https://dash.cloudflare.com → **R2** → crear un **bucket** → crear un
**API Token** (Access Key ID + Secret Access Key con acceso al bucket) → habilitar
un **dominio público** del bucket (r2.dev o un dominio propio).

| Valor | Variable | Dónde |
|---|---|---|
| Nombre del bucket | `AGORA_RECORDING_BUCKET` | Railway → **API** |
| Access Key ID | `AGORA_RECORDING_ACCESS_KEY` | Railway → **API** |
| Secret Access Key | `AGORA_RECORDING_SECRET_KEY` | Railway → **API** |
| Endpoint S3 (`https://<accountid>.r2.cloudflarestorage.com`) | `AGORA_RECORDING_ENDPOINT` | Railway → **API** |
| URL pública del bucket (para servir los replays) | `AGORA_RECORDING_PUBLIC_BASE` | Railway → **API** |

Activación: `AGORA_RECORDING_USE_MOCK=false` en el servicio **API**.

> Nota técnica: el `vendor`/`region` del storage de Agora para R2 se ajusta al
> conectar (Agora trata R2 como S3-compatible). El equipo de desarrollo lo
> verifica al cablear, ya que es configurable por variable
> (`AGORA_RECORDING_VENDOR`, `AGORA_RECORDING_REGION`).

**Costo:** Agora cobra por minuto de grabación; R2 cobra por almacenamiento y
operaciones (tiene un tier gratuito generoso). Ambos requieren método de pago
del cliente.

---

## Resumen de prioridad sugerida
1. **Windy** — 2 min, gratis, sin riesgo. Buen primer paso.
2. **Firebase** — gratis, ~10 min, desbloquea el push real.
3. **Agora Recording + R2** — la más costosa y laboriosa; dejar para el final.

Mientras tanto, **develop sigue 100% funcional en mock** para desarrollo y demos.
