---
sprint: 0
fecha: 2026-05-25
tema: skills
---

# Sprint 0 — Skills y agentes instalados

> Skills de Claude que orientan el código. Se consultan **antes** de construir features de su dominio en lugar de inventar patrones.

Sprint maestro: [[Sprint 0 - Setup (25 may)]]

---

## 📁 Dónde están

```
Globeliv/
├── .claude/skills/        ← skills del repo (commiteados en git)
│   ├── NOTICE.md
│   ├── accessibility-a11y/
│   ├── clean-architecture/
│   ├── drizzle-orm/
│   ├── framer-motion/
│   ├── monorepo/
│   ├── nestjs-clean-typescript/
│   ├── performance-optimization/
│   ├── postgresql-best-practices/
│   ├── redis-best-practices/
│   ├── security-best-practices/
│   ├── trpc/
│   └── zod-schema-validation/
│
└── .agents/               ← agentes nestjs-doctor (auditor)
    ├── nestjs-doctor/
    └── nestjs-doctor-create-rule/

~/.claude/skills/          ← skills user-level (NO en git)
├── ui-ux-pro-max/         (symlink)
├── design/                (symlink)
├── design-system/         (symlink)
├── ui-styling/            (symlink)
└── brand/                 (symlink)
```

---

## 🎨 Suite UI/UX (user-level, symlinks)

Origen: https://github.com/nextlevelbuilder/ui-ux-pro-max-skill — clonado en `~/.local/share/uiux-pro-max-skill/`.

| Skill | Cuándo usarlo |
|---|---|
| **`ui-ux-pro-max`** | **OBLIGATORIO antes de crear cualquier pantalla.** 50+ estilos, 161 paletas, 57 font pairings, 99 UX guidelines, 25 tipos de chart |
| `design` | Logos, identidad corporativa (CIP), mockups, slides, banners, íconos SVG, fotos sociales |
| `design-system` | Token architecture de 3 capas (primitive → semantic → component). Para evolucionar `packages/ui/src/tokens.css` |
| `ui-styling` | shadcn/Radix + Tailwind, themes, dark mode, layouts responsive |
| `brand` | Voice & tone, mensajería, identidad visual, compliance |

> Para actualizar: `cd ~/.local/share/uiux-pro-max-skill && git pull` — los symlinks recogen cambios.

## ⚙️ Backend (en repo, commiteados — Mindrally)

Origen: https://github.com/Mindrally/skills

| Skill | Cuándo usarlo |
|---|---|
| `nestjs-clean-typescript` | Patrones SOLID en módulos, services, controllers |
| `drizzle-orm` | Schemas, migrations, queries tipadas, joins |
| `trpc` | Procedures, middleware, inputs Zod, batching |
| `clean-architecture` | Decisiones de capas (controller → service → repo) |
| `performance-optimization` | Profiling de queries, bundle size, render perf |
| `redis-best-practices` | Estructuras de datos, TTLs, pub/sub |
| `postgresql-best-practices` | Indexes, particionado, EXPLAIN |
| `security-best-practices` | Helmet, CORS, secrets, OWASP |
| `monorepo` | Reglas de límites entre apps/packages |
| `zod-schema-validation` | Validación de inputs en boundaries |
| `framer-motion` | Animaciones, gestures, AnimatePresence, layout animations |
| `accessibility-a11y` | WCAG, ARIA, contraste, navegación por teclado |

## 🩺 nestjs-doctor (auditor)

Origen: https://github.com/RoloBits/nestjs-doctor

- Vive en `Globeliv/.agents/nestjs-doctor/` y en `dependencies` raíz del repo (`nestjs-doctor: ^0.4.32`)
- **Workflow:** después de tocar `apps/api`, correr antes del commit:
  ```bash
  pnpm exec nestjs-doctor .
  ```
- El agente complementario `nestjs-doctor-create-rule` se usa para añadir reglas custom al checker

---

## 📐 Reglas operativas que se escribieron en CLAUDE.md gracias a estos skills

Las tres no negociables que están enforced por código:

### 1. Escalabilidad como criterio único (§11.1)
- Cero trabajo síncrono pesado en request HTTP (>200ms → BullMQ)
- DB pools dimensionados (max 10 por instancia)
- Indexes ANTES que queries (mismo PR)
- Redis para hot-path (>1/seg)
- Background jobs idempotentes con `jobId` determinístico

### 2. HeroUI primero, custom último (§11.2)
- Default: todo componente desde HeroUI vía `@globeliv/ui`
- Excepción justificada: si HeroUI no lo tiene, custom **basado en primitivas HeroUI** + Tailwind
- Antes de crear custom: revisar https://www.heroui.com/docs/components
- Esta regla coincide con la auto-memoria del usuario: [[heroui_first]]

### 3. Framer Motion (§11.3)
- Ya está en bundle vía HeroUI peer dep
- Solo cuando añada valor real (transiciones, microinteracciones, hero)
- `>500ms` debe respetar `prefers-reduced-motion`

---

## 🔄 Setup tras fresh clone

```bash
# 1. Instalar deps
pnpm install

# 2. Setup nestjs-doctor (auto-instala en .agents/)
pnpm exec nestjs-doctor --init

# 3. UI UX Pro Max (uno-time por máquina)
git clone https://github.com/nextlevelbuilder/ui-ux-pro-max-skill \
  ~/.local/share/uiux-pro-max-skill

for skill in ui-ux-pro-max design design-system ui-styling brand; do
  ln -s ~/.local/share/uiux-pro-max-skill/.claude/skills/$skill \
    ~/.claude/skills/$skill
done
```

Origen autoritativo de esta lista: `Globeliv/.claude/skills/NOTICE.md`.
