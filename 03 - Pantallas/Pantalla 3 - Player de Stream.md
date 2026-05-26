# Pantalla 3 — Player de Stream

## Objetivo
**Inmersión total. Mínima distracción.**

## Estructura

1. **Video a pantalla completa**
   - Aspect ratio responsivo (16:9 horizontal, 9:16 vertical).
   - Botón **X** arriba a la izquierda para cerrar.
   - Botón **⚑** arriba a la derecha para reportar.

2. **Overlay inferior (sobre el video)**
   - Título del stream.
   - Metadata: `ubicación · avatar streamer · nombre · rating · 👁 contador`.

3. **Barra de acciones (debajo del video)**
   - Chat (toggle).
   - Reacción (animación al tocar).
   - **Pedir vista** ("¿Puedes mostrar X?").
   - **Propina** (abre modal).

4. **Propinas rápidas**
   - Botones inline: `$1 / $3 / $5 / $10 / Otro`.

5. **Chat en vivo**
   - Lado derecho en desktop, abajo en móvil.
   - Mensajes con nombre + texto.
   - El streamer destacado con color distinto.
   - Reacciones animadas que aparecen sobre el chat.
   - Input para enviar mensaje.

## Notas clave

- Las **propinas siempre están disponibles** (no las puede desactivar el streamer). Ver [[Reglas de UX que no se negocian]].
- El reporte abre un menú con razones predefinidas que alimenta [[Moderación Automática]].

---

### Relacionado
- [[Sistema de Propinas]]
- [[Flujo C - Recibir Propina]]
- [[Streaming en Vivo]]
- [[Moderación Automática]]
