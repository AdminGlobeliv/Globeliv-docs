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

## Detección automática en paralelo

8. Un sistema de **detección de contenido** analiza 1 frame cada 10 segundos.
9. Si detecta **nudity con confianza > 0.85** → advertencia.
10. Si detecta con **confianza > 0.95** → corte automático.

## Alerta al admin

11. El admin (`yegaf1@gmail.com`) recibe **alerta por email y Telegram** cuando se ejecuta un corte automático o una suspensión.

## Reglas clave

- Un usuario **solo puede reportar un stream una vez**.
- Las advertencias tienen **cooldown** para evitar spam de notificaciones al streamer.
- **3 cortes automáticos = ban permanente**.

---

### Relacionado
- [[Moderación Automática]]
- [[Pantalla 3 - Player de Stream]]
