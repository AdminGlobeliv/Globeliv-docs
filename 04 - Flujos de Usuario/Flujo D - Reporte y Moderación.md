# Flujo D — Reporte y moderación automática

## Pasos

1. El viewer **ve contenido inapropiado** en un stream.
2. Toca el botón **⚑ Reportar**.
3. **Selecciona razón**:
   - Nudity
   - Violencia
   - Acoso
   - Spam
   - Ilegal
   - Menor de edad
   - Otro
4. El sistema **cuenta los reportes** del stream en los últimos 5 minutos.
5. **Si llega a 3 reportes** → advertencia automática al streamer.
6. **Si llega a 5 reportes** → corte automático del stream.
7. **Si llega a 8+ reportes** → corte + suspensión de 7 días.

## Detección automática en paralelo (NSFW)

8. Un sistema de **detección de contenido** (NSFW.js, open-source y **gratis**) analiza 1 frame cada 10 segundos.
9. Si detecta **nudity con confianza > 0.85** → advertencia.
10. Si detecta con **confianza > 0.95** → corte automático.

> **Nota (implementación):** el frame se toma de la **miniatura que el host ya sube** al feed cada ~7s (Redis), así que la detección **NO necesita Agora Cloud Recording ni R2** — es gratis. Tradeoff: al venir del navegador del host, un host técnico podría falsear su frame; lo mitigan los reportes de usuarios (que sí son del servidor). La versión a prueba de manipulación (captura server-side de Agora) es un upgrade de pago futuro.

## Alerta al admin

11. El admin (`adminglobeliv@gmail.com`) recibe **alerta por email y Telegram** cuando se ejecuta un corte automático o una suspensión.

## Reglas clave

- Un usuario **solo puede reportar un stream una vez** (los 5 reportes para el corte son de **5 personas distintas**, no 5 del mismo usuario).
- Las advertencias tienen **cooldown** para evitar spam de notificaciones al streamer.
- **Escalamiento de cuenta por cortes acumulados:** 1º solo se registra · 2º suspensión 7 días · 3º suspensión 30 días · **4º corte = ban permanente**. (Antes esta nota decía "3 cortes = ban"; era un error — el código y [[Moderación Automática]] usan 4º.)

---

### Relacionado
- [[Moderación Automática]]
- [[Pantalla 3 - Player de Stream]]
