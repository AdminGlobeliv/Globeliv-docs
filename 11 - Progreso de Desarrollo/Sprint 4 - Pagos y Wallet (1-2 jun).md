---
sprint: 4
nombre: Pagos y Wallet (Stripe Connect + propinas 90/10)
fecha_inicio: 2026-06-01
fecha_fin: 2026-06-02
estado: code-complete · DIFERIDO (corre en mock)
duración: 1 día de build (1 jun) + viaja en el deploy de S5 (2 jun)
---

# Sprint 4 — Pagos y Wallet ⏸️ (code-complete, diferido)

> _"Que un viewer pueda dar propina con tarjeta a un streamer en vivo, el 90% le llegue, y el streamer vea sus ganancias y pueda retirar."_

**Estado:** **código 100% completo y verificado en mock/estático (2026-06-01)**, pero **DIFERIDO** ese mismo día: el usuario **no puede crear cuenta Stripe** (ni de prueba), casi seguro por el **límite de Stripe en Perú**. Se decidió dejar la integración de pagos "para el final" y saltar a [[Sprint 5 - Misiones (2-3 jun)|Sprint 5]]. El código S4 viaja en el commit `f5659f1` (2026-06-02, junto con S5) y **corre en modo mock en develop** — propinas deshabilitadas en la UI, sin riesgo. Migración `0006` aplicada a la BD de develop.

---

## 🎯 Objetivo (ROADMAP §Sprint 4 + spec §Flujo C, §13)

A partir de este sprint **el producto puede ganar dinero**. **Definición de terminado:** "un viewer puede dar propina a un streamer en vivo, el 90% le llega y el streamer ve sus ganancias".

| # | Entregable | Estado |
|---|---|---|
| 1 | **Onboarding de pagos** (KYC Stripe Connect) | ✅ código (mock) |
| 2 | **Propinas** con tarjeta (90/10) | ✅ código (mock) |
| 3 | **Wallet / ganancias** del streamer | ✅ código (mock) |
| 4 | **Retiros** | ✅ vía Express Dashboard de Stripe |
| 5 | **Comisión automática 90/10** | ✅ destination charge |

---

## 🧱 Decisiones de arquitectura (confirmadas con el usuario, 2026-06-01)

1. **Stripe Connect Express + destination charges.** `PaymentIntent` con `application_fee_amount` (10%) + `transfer_data.destination` = cuenta del streamer → el 90% cae directo en su balance Stripe. KYC/onboarding hosteado por Stripe.
2. **Payouts nativos de Stripe — NO custodiamos fondos.** "Wallet / Retirar" = link al Express Dashboard. La card "Ganancias" se calcula desde nuestro ledger (`tips` / `transactions`), no de un balance interno.
3. **Solo propinas en S4.** El pay-per-view (ACCESO $1/$3/$5) se difiere — el enum `monetization_level` ya existe; el gate (rating ≥4.5 y 20 streams) se suma después sin re-arquitectura.
4. **Idempotencia del webhook:** `stripePaymentIntentId` UNIQUE + dedup por `event.id` en Redis (SET NX TTL 24h) → un replay de Stripe nunca cobra dos veces (principio #7).

---

## 🛠 Lo que se construyó (por fase)

Build organizado en fases **F0→F6**. Todo code-complete + verificado estático/mock el 2026-06-01.

### F0 — Infra + schema
- Migración **`0006`**: tablas **`tips`** (FKs cascade, `amount`/`platformFee`/`streamerAmount` numeric, `stripePaymentIntentId` UNIQUE, índices) y **`transactions`** (ledger de doble asiento: `type` enum define dirección, `stripeReference`). Campos nuevos en `users`: `stripe_account_id`, `stripe_customer_id`, `stripe_charges_enabled`, `stripe_payouts_enabled` (cache del estado de onboarding, sincronizado por webhook `account.updated`). Enums `tip_status` y `transaction_type`.
- SDK `stripe@22.2.0` en `apps/api`, **apiVersion pinneada `2026-05-27.dahlia`**. `StripeService` (cliente real o null en mock, `computeSplit(cents)`) + singleton `getStripeServiceInstance()` cableado al context tRPC.
- Env vars con `superRefine` (si `STRIPE_USE_MOCK=false` exige keys): `STRIPE_USE_MOCK` (default true), `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PLATFORM_FEE_BPS` (1000=10%), `STRIPE_MIN_TIP_CENTS` (50), `STRIPE_CURRENCY`. Documentadas en `.env.example`.

### F1 — Onboarding Connect
- `StripeService`: `createConnectAccount` (Express), `createAccountLink` (onboarding), `getAccountStatus`, `createDashboardLink` (= "Retirar").
- Router `payments`: `createOnboardingLink`, `accountStatus` (refresca y cachea), `dashboardLink`.
- UI: `stripe-payouts-card.tsx` en Settings con 3 estados (sin configurar → verificando → activo). Reemplazó el placeholder "Sprint 4".

### F2 — Propinas backend (90/10)
- `StripeService`: `createCustomer`, `createEphemeralKey` (tarjetas guardadas), `createTipPaymentIntent` (destination charge idempotente), `constructWebhookEvent` (HMAC).
- Router `tips.create`: valida mock/min/stream-live/no-self-tip/streamer-con-payouts, `computeSplit`, PaymentIntent idempotente, INSERT tip `pending`.
- Contrato realtime: evento `tip:received` agregado a `realtime.ts`.

### F3 — Webhook HMAC + ledger
- `stripe-webhook.ts`: verifica firma con raw body, dedup por `event.id` en Redis, maneja `payment_intent.succeeded` / `payment_failed` / `account.updated`. (El raw body ya estaba montado desde S2.)
- `tips-store.ts`: `finalizeTipPaid` (UPDATE WHERE `status='pending'` RETURNING atómico + doble asiento en transacción), total de propinas por stream en Redis. Emite `tip:received` a la room del stream y del streamer.

### F4 — Propinas UI (player)
- SDKs web `@stripe/stripe-js` + `@stripe/react-stripe-js`. `tip-modal.tsx` (2 fases: monto $1/$3/$5/$10/Otro → pagar con PaymentElement theme night). `floating-tips.tsx` (avisos dorados), `use-stream-tips.ts` (escucha `tip:received` → flotantes + total + "ka-ching" Web Audio, solo suena al destinatario). Botón Propina + `TipTotalChip` cableados en el watch (host y viewer ven flotantes; viewer tiene modal).

### F5 — Wallet / ganancias
- `payments.earnings` (mes vía `date_trunc` + lifetime) + `payments.transactions` (ledger paginado). `ProfileEarningsCard` con datos reales. `transactions-modal.tsx` desde Settings. "Retirar" = "Abrir panel de Stripe" (`dashboardLink`).

---

## 🔬 Verificación (2026-06-01)

- `typecheck` 10/10 + `lint` api/web verdes + `next build` web 10/10 páginas.
- API arranca limpio en mock con todo el código S4. Smoke test: webhook responde 503 (mock), `payments.accountStatus`/`tips.create` exigen auth (UNAUTHORIZED).
- **NO se pudo hacer E2E real** porque requiere keys Stripe test (`sk_test_` + `pk_test_`) y activar Connect — bloqueado por el límite de Perú.

---

## ⏸️ Por qué quedó DIFERIDO (2026-06-01)

El usuario **no puede crear cuenta Stripe** (anticipado: límite de Perú). Decisión: dejar pagos para el final y pasar a S5 (que NO depende de proveedor externo).

- El código S4 corre en **mock**: propinas deshabilitadas en UI con tooltip, webhook → 503, `tips.create` → `PRECONDITION_FAILED`. Cero riesgo.
- **Al retomar pagos, lo más probable es usar un proveedor alternativo para Perú** (Mercado Pago / Culqi / dLocal) en vez de Stripe. El código de propinas (split 90/10, ledger, realtime, UI) se reutiliza casi todo — solo cambia el "enchufe" (`StripeService` → un `PaymentProvider`).
- Stripe CLI instalado (`~/.local/bin/stripe`) para cuando haya cuenta.

---

## 🚀 Deploy (en mock, junto con S5 — 2026-06-02)

- El código S4 **no se desplegó solo**: viajó en el commit `f5659f1` (S4+S5) al desplegar S5. Migración `0006` aplicada a la Postgres de Railway develop junto con la `0007`.
- En develop corre en **mock** (sin `STRIPE_USE_MOCK=false`, sin keys). Propinas off en UI — correcto.

---

## 🧾 Deuda técnica / pendientes

- **Bloqueante del cierre real:** keys Stripe test + activar Connect → probar onboarding + propina real (tarjeta 4242) + webhook (`stripe listen`) → ledger + `tip:received` en vivo. **Diferido indefinidamente hasta resolver proveedor para Perú.**
- **Pay-per-view (ACCESO):** diferido; enum listo, falta el gate y la UI de compra.
- **Bug latente de env:** `optionalSecret()` ya arregla `KEY=` vacío en las vars Stripe; el mismo patrón debería aplicarse a `AGORA_APP_ID`/`CERTIFICATE` si alguna vez molesta.
- **Revocar el `RAILWAY_TOKEN` temporal** usado para aplicar la migración 0006 en develop.

---

## 🔗 Conexiones

- Viene de: [[Sprint 3 - Realtime (30 may - 1 jun)]]
- Estado en memoria: [[globeliv-s4-payments-state]]
- Pantallas que toca: [[Pantalla 3 - Player de Stream]] (propinas), [[Pantalla 6 - Perfil]] (ganancias), [[Pantalla 7 - Configuración]] (payouts)
- Sigue: [[Sprint 5 - Misiones (2-3 jun)]] (se priorizó por no depender de proveedor externo).
