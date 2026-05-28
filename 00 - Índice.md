# 🌍 GlobeLiv — Índice maestro

> _"See the world as it happens"_
>
> Plataforma de streaming en vivo geolocalizado donde las personas piden ver lugares del mundo y otras los transmiten en tiempo real.

Este es el **mapa de toda la documentación**. Empieza por aquí.

---

## 🚀 Si es tu primera vez

Léelo en este orden:

1. [[Qué es GlobeLiv]] — qué estamos construyendo.
2. [[El Diferenciador]] — qué nos hace únicos.
3. [[Filosofía del Producto]] — cómo decidimos.
4. [[Por qué va a funcionar]] — la oportunidad de mercado.
5. [[Lección de Periscope]] — lo que no podemos repetir.

Después, mira [[Reglas de UX que no se negocian]] y la sección **Pantallas**.

---

## 📦 Producto

- [[Qué es GlobeLiv]]
- [[El Diferenciador]]
- [[Por qué va a funcionar]]
- [[Filosofía del Producto]]
- [[Lección de Periscope]]

## 🎨 Identidad Visual

- [[Logo y Símbolos]]
- [[Paleta de Colores]]
- [[Tipografía e Iconos]]

## 📱 Las 7 Pantallas del MVP

- [[Pantalla 1 - Onboarding]] — sin registro, mostrar valor inmediato.
- [[Pantalla 2 - Home Explorar]] — donde el usuario descubre streams.
- [[Pantalla 3 - Player de Stream]] — inmersión total.
- [[Pantalla 4 - Go Live]] — cero fricción para transmitir.
- [[Pantalla 5 - Misiones]] — el corazón del diferenciador.
- [[Pantalla 6 - Perfil]] — confianza y gamificación.
- [[Pantalla 7 - Configuración]] — settings y wallet.

## 🔄 Flujos de Usuario

- [[Flujo A - Primera Visita]] — del cero al primer stream.
- [[Flujo B - Crear una Solicitud]] — el flujo más importante.
- [[Flujo C - Recibir Propina]] — la monetización en acción.
- [[Flujo D - Reporte y Moderación]] — cómo se cuida la comunidad.

## ⚙️ Sistemas Clave

### Core de la experiencia
- [[Streaming en Vivo]]
- [[Pasaporte del Explorador]]
- [[Moderación Automática]]
- [[Webcams de Respaldo]]
- [[Notificaciones Push]]
- [[Replays]]
- [[Eventos Programados]]
- [[Feed Personalizado]]
- [[Sistema de Seguir]]
- [[Sistema de Ratings]]

### Monetización
- [[Sistema de Propinas]]
- [[Pay-per-view]]
- [[Sistema de Misiones]]
- [[Wallet y Retiros]]

## 🛡 Reglas de oro

- [[Reglas de UX que no se negocian]]

## 🗓 Roadmap

- [[Sprint 0 - Setup]]
- [[Sprint 1 - Auth y Usuarios]]
- [[Sprint 2 - Streaming Core]]
- [[Sprint 3 - Interacción Social]]
- [[Sprint 4 - Propinas y Wallet]]
- [[Sprint 5 - Misiones]]
- [[Sprint 6 - Retención y Pasaporte]]
- [[Sprint 7 - Moderación]]
- [[Sprint 8 - Pulido y Lanzamiento]]

## 🔐 Control y Seguridad

- [[Propiedad y Accesos]]
- [[Reglas de Seguridad]]
- [[Qué hacer si algo falla]]

## 💼 Negocio y Lanzamiento

- [[Modelo Económico]] — comisiones, mínimos, fuentes de ingreso.
- [[Métricas del MVP]] — KPIs y la estrella del norte.
- [[Estrategia de Lanzamiento]] — cómo resolver el cold start.
- [[Soporte al Usuario]] — qué hacer cuando la moderación automática no alcanza.
- [[Plan Post-MVP]] — qué viene después de la semana 16.
- [[Legal Mínimo]] — términos, privacidad, compliance.

## ✅ Antes de empezar

- [[Checklist para empezar]]

## 🛠 Progreso de Desarrollo (build log)

Bitácora cronológica de **qué se ha construido** sprint a sprint, con fechas, commits y decisiones. Complemento técnico a este índice de producto.

- [[00 - Índice de Progreso]] — entrada principal a la bitácora
	- [[Sprint 0 - Setup (25 may)]] — monorepo, packages, CI ✅
	- [[Sprint 1 - Auth y Usuarios (26-27 may)]] — auth, deploys, OAuth ✅
	- [[Sprint 2 - Streaming Core (27-28 may)]] — streams, Agora 🚧

## 🏗 Arquitectura Técnica

Referencia atemporal de **cómo funciona el sistema hoy**. Diagramas Mermaid, flujos end-to-end, decisiones (ADRs), deploy.

- [[00 - Índice de Arquitectura]] — entrada con mapa de la sección
	- [[Visión General del Sistema]] — diagrama maestro 3 capas
	- [[Stack y Versiones]] · [[Topología de Despliegue]]
	- [[Flujo Frontend (Next.js)]] · [[Flujo Backend (NestJS)]] · [[Flujo Workers (BullMQ)]]
	- [[Flujo End-to-End — Auth]] · [[Flujo End-to-End — Streaming]]
	- [[Modelo de Datos]] · [[Seguridad y Auth]]
	- [[Decisiones de Arquitectura (ADRs)]]

---

## 🧠 Mapa de conexiones clave

```
Qué es GlobeLiv
   └── El Diferenciador ─┬──► Sistema de Misiones ──► Pantalla 5 - Misiones
                         │                            └─► Flujo B - Crear una Solicitud
                         └──► Lección de Periscope

Filosofía del Producto ──► Reglas de UX que no se negocian
                            ├─► Pantalla 1 - Onboarding (regla #1, #2)
                            ├─► Sistema de Propinas (regla #4)
                            └─► Notificaciones Push (regla #3)

Streaming en Vivo ──► Pantalla 4 - Go Live
                  └─► Pantalla 3 - Player de Stream ──┬─► Sistema de Propinas
                                                       ├─► Pay-per-view
                                                       └─► Moderación Automática
                                                            └─► Flujo D - Reporte y Moderación

Sistema de Ratings ──► Pay-per-view (desbloqueo)
                   └─► Feed Personalizado (ordena)

Sistema de Seguir ──► Feed Personalizado (prioridad #1)
                  └─► Notificaciones Push (alta prioridad)

Pasaporte del Explorador ──► Pantalla 6 - Perfil
                          └─► Notificaciones Push

Modelo Económico ─┬─► Sistema de Propinas (10%)
                  ├─► Pay-per-view (20%)
                  ├─► Sistema de Misiones (15%)
                  └─► Wallet y Retiros

Métricas del MVP ──► Estrategia de Lanzamiento ──► Plan Post-MVP
                                              └──► Soporte al Usuario
                                                    └──► Legal Mínimo
```

---

## 📝 Notas sobre esta documentación

- Es la **traducción narrativa** del documento técnico `GLOBELIV_Especificacion.md`.
- Aquí solo están las **historias y conceptos** — no hay código, no hay esquemas de base de datos, no hay endpoints.
- El objetivo: **guiar las decisiones de producto** sin perderse en detalles técnicos.
- Cada vez que aprendas algo nuevo del proyecto, **agrégalo como una nota más** y conéctala con `[[wikilinks]]`.

---

### 🔗 Contacto
- **Fundador y Product Owner:** `yegaf1@gmail.com`
- **Dominio:** [globeliv.com](https://globeliv.com)
