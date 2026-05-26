# Feed Personalizado

El feed es **lo que el usuario ve al abrir la app**. Su orden no es aleatorio — tiene una jerarquía explícita.

## Prioridad del contenido

Cuando se arma el feed para un usuario, se llena en este orden:

| # | Tipo | Cuándo aparece |
|---|---|---|
| 1 | **Streamers seguidos en vivo** | Siempre primero. Ver [[Sistema de Seguir]]. |
| 2 | **Streams populares en vivo** | Si el feed no se llenó con seguidos. |
| 3 | **Webcams públicas** cerca del usuario | Si aún hay espacio. Ver [[Webcams de Respaldo]]. |
| 4 | **Replays destacados** | Si todavía no se llegó al mínimo de items. |

Objetivo: **el feed nunca está vacío**.

## Mínimo de items

El feed apunta a entre **10 y 15 items** visibles. Si con 1+2 no se llega, se completa con 3 y 4.

## Filtros sobre el feed

El usuario puede aplicar filtros desde [[Pantalla 2 - Home Explorar]]:
- En vivo / Gratis / Popular / Cerca / Atardeceres / Vida nocturna / Eventos.

Estos filtros **reordenan el feed**, no lo reemplazan.

## Personalización adicional (entrada)

Lo que conoce el sistema de cada usuario para mejorar el feed:
- A quién sigue.
- Categorías preferidas (declaradas en onboarding).
- Ubicación.
- Idioma preferido.

## Por qué este orden importa

- Si lo primero que ve el usuario son **personas que conoce o que siguió** → siente que el producto es "suyo".
- Si lo primero son **streams populares** → siente que hay vida en la plataforma.
- Si lo primero son **webcams genéricas** → siente que la plataforma está vacía y se va.

Por eso las webcams **nunca aparecen primero, aunque haya pocas opciones propias**.

---

### Relacionado
- [[Sistema de Seguir]]
- [[Webcams de Respaldo]]
- [[Replays]]
- [[Pantalla 2 - Home Explorar]]
- [[Sistema de Ratings]]
