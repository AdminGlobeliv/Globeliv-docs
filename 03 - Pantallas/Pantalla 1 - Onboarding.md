# Pantalla 1 — Onboarding / Splash (sin registro)

## Objetivo
Que el usuario **vea contenido real antes de pedirle registrarse**.

## Comportamiento

- Al abrir globeliv.com por primera vez, mostrar **inmediatamente un stream en vivo a pantalla completa**.
- El stream mostrado es el de **mayor cantidad de viewers activos** en ese momento.
- Si no hay streams en vivo con más de 5 viewers, mostrar el **replay con mayor peak de viewers** de las últimas 24 horas.
- Overlay sutil en la parte inferior: _"Esto está pasando ahora mismo en [ubicación]. Toca para explorar más."_
- Después de **5 segundos**, mostrar un botón flotante: _"Explorar más streams"_.
- Solo después de **30-45 segundos** de exploración, mostrar el prompt de registro.

## Lo que NO se hace

- ❌ Mostrar pantalla de registro primero.
- ❌ Mostrar tutorial de pasos.
- ❌ Mostrar mapa vacío.

## Por qué importa

Esta pantalla materializa la [[Filosofía del Producto]]: el usuario ve valor antes de que le pidan algo.

---

### Relacionado
- [[Flujo A - Primera Visita]]
- [[Reglas de UX que no se negocian]]
- [[Webcams de Respaldo]]
- [[Replays]]
