# Moderación Automática

GlobeLiv **modera 24/7 sin intervención humana**. El admin (`yegaf1@gmail.com`) solo es alertado cuando algo importante pasa.

## Tres fuentes de moderación

### 1. Reportes de usuarios
Cuando los viewers tocan ⚑ Reportar en un stream:

- **3 reportes** en 5 min → advertencia automática al streamer.
- **5 reportes** en 5 min → corte automático del stream.
- **8+ reportes** en 5 min → corte + suspensión de 7 días.
- Un usuario solo puede reportar un stream **una vez**.

### 2. Detección automática de contenido (NSFW)
Un worker analiza un frame del stream cada 10 segundos:

- **Confianza > 0.85** (nudity) → advertencia.
- **Confianza > 0.95** → corte automático.

### 3. Filtro de chat
Palabras prohibidas se bloquean automáticamente. Spam (mismo mensaje 3 veces, 15+ msgs/min) también.

## Escalamiento de cuenta

Cuando un streamer acumula cortes automáticos:

| Cortes | Consecuencia |
|---|---|
| 1° | Solo se registra |
| 2° | Suspensión 7 días |
| 3° | Suspensión 30 días |
| 4° | **Ban permanente** |

## Alertas al admin

Cuando ocurre un corte automático, suspensión o ban, el admin recibe:
- **Email** a `yegaf1@gmail.com`.
- **Mensaje de Telegram** al chat configurado.

## Cooldowns y gracias

- Las advertencias tienen **cooldown** para evitar spam.
- Después de la 2ª advertencia, el streamer tiene **30 segundos de gracia** antes del corte.

## Por qué importa

- Permite escalar la moderación **sin contratar gente**.
- Protege al producto del riesgo legal y reputacional desde el día 1.
- Las **reglas son configurables** (umbrales, ventanas de tiempo) sin necesidad de redeploy.

---

### Relacionado
- [[Flujo D - Reporte y Moderación]]
- [[Pantalla 3 - Player de Stream]]
- [[Qué hacer si algo falla]]
