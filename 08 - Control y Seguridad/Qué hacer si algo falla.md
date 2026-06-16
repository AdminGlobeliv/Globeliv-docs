# Qué hacer si algo falla

Plan de acción rápido para los **4 incidentes más comunes**.

## 🔥 Backend caído

1. Revisar **logs** en el dashboard de hosting.
2. Si es **error de código** → revertir al último commit estable.
3. Si es **error de base de datos** → verificar las migraciones.
4. Si es **error de servicio externo** → revisar el status del proveedor (streaming, pagos, notificaciones).
5. **Notificar al fundador (`adminglobeliv@gmail.com`) inmediatamente.**

## 📺 Stream no carga

1. Verificar que el ID de la app de streaming esté correcto en el frontend.
2. Verificar que el **token** no esté expirado.
3. Revisar los logs del proveedor de streaming.

## 💳 Pago no procesa

1. Verificar las **firmas de los webhooks**.
2. Revisar logs del proveedor de pagos.
3. Confirmar que el streamer **completó su KYC**.

## 🐛 Usuario reporta bug crítico

1. **Reproducir localmente.**
2. Crear issue en GitHub.
3. Si **impide usar la app** → hotfix inmediato.
4. Si **no es crítico** → programar para el siguiente sprint.

## Principio general

> No improvises en producción. **Documenta, revierte, comunica.**

---

### Relacionado
- [[Propiedad y Accesos]]
- [[Reglas de Seguridad]]
- [[Moderación Automática]]
