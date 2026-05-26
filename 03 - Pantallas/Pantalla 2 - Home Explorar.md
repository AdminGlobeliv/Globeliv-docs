# Pantalla 2 — Home / Explorar

## Objetivo
Mostrar contenido relevante en **menos de 3 segundos**.

## Estructura visual (de arriba a abajo)

1. **Header sticky**
   - Logo GlobeLiv (izquierda).
   - Contador en vivo: punto rojo pulsante + texto _"4,283 en vivo"_ (derecha).

2. **Barra de búsqueda**
   - Placeholder: _"Busca cualquier lugar o evento..."_
   - Icono de lupa a la izquierda.

3. **Filtros horizontales** (scroll horizontal)
   - En vivo (activo por defecto)
   - Gratis
   - Popular
   - Cerca
   - Atardeceres
   - Vida nocturna
   - Eventos

4. **Sección "Transmisiones en vivo"**
   - Título + link _"Ver mapa →"_
   - Cards verticales (una por stream activo).

5. **Botón flotante "Go Live"**
   - Circular, color rojo, con sombra glow.
   - Centrado en la parte inferior.
   - Solo visible para usuarios registrados.

## Anatomía de cada card de stream

- **Thumbnail (16:9)** con preview del stream.
- **Badges sobre el thumbnail:**
  - "EN VIVO" (rojo).
  - 👁 contador de viewers (negro semitransparente).
  - "GRATIS" / "$1 USD" / "Propinas" (verde / dorado / azul).
- **Debajo del thumbnail:**
  - Título del stream (1 línea, se trunca si es largo).
  - Línea de metadata: `ubicación · avatar streamer · nombre · rating`.

---

### Relacionado
- [[Pantalla 3 - Player de Stream]]
- [[Filosofía del Producto]]
- [[Reglas de UX que no se negocian]]
