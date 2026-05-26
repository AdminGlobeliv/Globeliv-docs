# Pantalla 5 — Misiones / Solicitudes

## Objetivo
**Doble propósito:** el viewer crea solicitudes, el streamer ve las disponibles cerca.

Esta pantalla materializa [[El Diferenciador]] del producto.

## Estructura

1. **Header**
   - Título: _"Misiones"_.
   - Subtítulo: _"Personas quieren ver estos lugares. Transmite y gana dinero."_

2. **Botón crear solicitud** (con borde punteado)
   - _"+ Pedir ver un lugar"_.
   - Ocupa todo el ancho.
   - Al tocar abre un modal con los campos:
     - Ubicación (autocompletado con Google Places).
     - Descripción opcional.
     - Recompensa opcional.

3. **Lista de solicitudes activas cerca**
   - Cards con la anatomía descrita abajo.

## Anatomía de cada card de solicitud

- **Ubicación:** _"📍 Plaza de Armas, Cusco"_.
- **Badge de recompensa** dorado: _"$5 USD"_ — o _"💝 Solo propinas"_ si no tiene recompensa.
- **Descripción:** _'"Quiero ver cómo está la plaza ahora, ¿hay mucha gente?"'_.
- **Metadata:** _"⏱ Hace 12 min · 2.3 km de ti · por @TravelMike"_.
- **Botón verde:** _"ACEPTAR Y TRANSMITIR →"_.

## Notas

- El streamer solo ve solicitudes en un **radio cercano** a su ubicación.
- El **primero en aceptar** gana la misión.
- Si nadie acepta en 24 horas, el dinero se devuelve al viewer.

---

### Relacionado
- [[Sistema de Misiones]]
- [[Flujo B - Crear una Solicitud]]
- [[El Diferenciador]]
