# Eventos Programados

A diferencia de los streams espontáneos, los **eventos programados** se anuncian con antelación y se asignan a un streamer específico.

## Qué tipo de eventos

- **Amaneceres** (sunrise) en lugares icónicos.
- **Atardeceres** (sunset).
- **Festivales** y celebraciones.
- **Mercados** de noche o de madrugada.
- **Eventos de ciudad** (carnaval, fuegos artificiales, partidos).
- **Naturaleza** (auroras boreales, eclipses, migraciones).

## Cómo se crean

- Inicialmente los crea el equipo de GlobeLiv o el admin desde el panel.
- Cada evento tiene **ubicación + fecha/hora + categoría + streamer asignado**.

## Cómo se muestran al usuario

Los eventos aparecen:
- En la pestaña **"Eventos"** dentro de los filtros del [[Pantalla 2 - Home Explorar]].
- Como **notificación push** 15 minutos antes del evento (si el usuario activó recordatorios).

## Regla clave: zona horaria

**Siempre se muestran dos horarios**:

> _"Mañana a las 9:30 PM hora tuya (5:30 AM hora local en Santorini)"_

El usuario nunca debe tener que **calcular mentalmente la diferencia horaria**. Ver [[Reglas de UX que no se negocian]] #6.

## Estado de un evento

| Estado | Significa |
|---|---|
| upcoming | Programado, aún no empezó |
| live | Está sucediendo ahora |
| completed | Ya terminó, puede haber replay |
| cancelled | Se canceló |

## Por qué importa para el MVP

- Le dan al usuario una **razón para volver en un momento específico** (no solo cuando recuerda).
- Crean **expectativa compartida** entre la comunidad ("todos vamos a ver el amanecer en Santorini el jueves").
- Son contenido **predecible** que el equipo puede curar y promocionar.

---

### Relacionado
- [[Replays]]
- [[Notificaciones Push]]
- [[Feed Personalizado]]
- [[Pantalla 2 - Home Explorar]]
- [[Reglas de UX que no se negocian]]
