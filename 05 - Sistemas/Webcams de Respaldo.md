# Webcams de Respaldo

## El problema que resuelve

Al principio del proyecto **no habrá suficientes streamers en vivo** para llenar el feed. Si el usuario abre la app y ve pocas opciones, se va.

## La solución

Integrar **webcams públicas** (de un proveedor externo como Windy) que se muestran **solo cuando el feed propio es insuficiente**.

## Prioridad del feed

El feed personalizado se llena en este orden:

1. **Streamers que sigues** en vivo.
2. **Streams populares** en vivo.
3. **Webcams públicas** cerca del usuario _(si el feed sigue corto)_.
4. **Replays destacados** _(si aún hay espacio)_.

## Reglas obligatorias

- Cuando se muestre una webcam, **debe aparecer crédito visible** al proveedor: _"Powered by Windy.com"_.
- Las webcams están marcadas claramente como **diferentes** de los streams de personas (no son interactivos, no se les puede dar propina).

## Importante

Esto es **temporal estratégico**. A medida que crece la base de streamers, las webcams van perdiendo importancia. Es la solución al "cold start" del marketplace.

---

### Relacionado
- [[Pantalla 1 - Onboarding]]
- [[Filosofía del Producto]]
- [[Replays]]
