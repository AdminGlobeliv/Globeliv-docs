# Legal Mínimo

Lo que **debe estar listo antes del lanzamiento público** para no tener problemas serios.

> ⚠️ Esta nota es una **guía conceptual**. La redacción y validación legal la hace **un abogado especializado**, no el equipo de producto.

## Documentos obligatorios

### 1. Términos y Condiciones
- Qué se puede y no se puede transmitir.
- Reglas de moderación (ver [[Moderación Automática]]).
- Política de comisiones (ver [[Modelo Económico]]).
- Política de cuentas suspendidas o baneadas.
- Quién es dueño del contenido (el streamer mantiene la propiedad, GlobeLiv tiene licencia de uso).

### 2. Política de Privacidad
- Qué datos se recolectan (email, ubicación, cámara, micrófono, etc.).
- Para qué se usan.
- Quiénes son los terceros (streaming, pagos, notificaciones, analytics).
- Derechos del usuario (acceso, rectificación, borrado).
- Cumplimiento de **GDPR** (UE), **LGPD** (Brasil), **Ley Perú 29733**.

### 3. Política de Cookies / Tracking
- Banner de consentimiento si hay usuarios en UE.
- Tipos de cookies (esenciales, analíticas, marketing).

### 4. Política de Reembolsos
- Casos automáticos (stream < 2 min, misión expirada).
- Casos por revisión humana.
- Casos que no proceden.

### 5. Política de Pagos a Streamers
- Cómo se calcula el saldo.
- Tiempos de retiro.
- Responsabilidad fiscal (el streamer es responsable de declarar sus ingresos).

## Temas críticos

### Menores de edad
- **Edad mínima para registrarse:** 18 años.
- Si se detecta un menor transmitiendo → corte inmediato + ban + alerta a autoridades si aplica.
- En reportes existe la razón explícita **"menor de edad"** que escala automáticamente.

### Contenido en directo
- Disclaimer: GlobeLiv **no es responsable** del contenido generado por usuarios.
- Compromiso: **modera activamente** según las reglas publicadas.

### Datos de ubicación
- El streamer **decide** si publica su ubicación exacta (tag de ciudad) o aproximada.
- Nunca se muestra dirección exacta del streamer.
- Los viewers no pueden ver dónde está físicamente otro viewer.

### Internacional
- Algunos países regulan estrictamente las apps de streaming (China, Rusia, Irán). Se bloquean preventivamente.
- En la UE, derecho al olvido total (no solo desactivación de cuenta).

## Compliance de pagos

- Los proveedores de pago exigen **KYC** del streamer antes de transferir dinero.
- GlobeLiv **no almacena** datos de tarjeta — se delega al proveedor (PCI-DSS).
- Los webhooks de pagos se **verifican con firma** para evitar fraude.

## Quién firma qué

- Cada usuario al registrarse acepta los Términos y la Política de Privacidad.
- Cada streamer al activar pagos acepta condiciones adicionales del proveedor de pagos.
- Cada empleado / colaborador firma **NDA** y **cesión de propiedad intelectual**.

## Antes de lanzar al público

- [ ] Términos validados por un abogado.
- [ ] Política de privacidad validada.
- [ ] Páginas legales accesibles desde el footer y desde [[Pantalla 7 - Configuración]].
- [ ] Mecanismo de "borrado de cuenta" funcional.
- [ ] Mecanismo de "descarga de mis datos" funcional (GDPR).
- [ ] Soporte al usuario listo para manejar peticiones legales. Ver [[Soporte al Usuario]].

---

### Relacionado
- [[Propiedad y Accesos]]
- [[Reglas de Seguridad]]
- [[Moderación Automática]]
- [[Modelo Económico]]
- [[Soporte al Usuario]]
- [[Wallet y Retiros]]
