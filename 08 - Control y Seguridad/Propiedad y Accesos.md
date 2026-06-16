# Propiedad y Accesos

Reglas claras sobre **quién es dueño de qué** y **qué accesos tiene cada persona**.

## Principio fundamental

**TODAS las cuentas son creadas y propiedad del fundador (`adminglobeliv@gmail.com`).** El developer recibe acceso como colaborador con permisos limitados.

## Cuentas críticas (propiedad del fundador)

| Servicio | Quién es dueño | Qué recibe el developer |
|---|---|---|
| GitHub | Cuenta personal del fundador | Acceso "Developer" (push, sin borrar) |
| Streaming (Agora) | Cuenta del fundador | App ID — **nunca** App Certificate |
| Pagos (Stripe) | Cuenta del fundador | Publishable Key — **nunca** Secret Key |
| Firebase | Proyecto del fundador | Config del frontend |
| AWS | Cuenta del fundador | IAM user con permisos limitados |
| Dominio globeliv.com | Registrado a nombre del fundador | — |
| Email empresa | `admin@globeliv.com` gestionado por fundador | — |
| Hosting (Railway/Vercel) | Cuenta del fundador | Acceso "Member" (logs, sin billing) |

## Reglas obligatorias

- **2FA activado** en todas las cuentas críticas, con el celular del **fundador**, no del developer.
- **Gestor de contraseñas** (Bitwarden o 1Password). Todas las credenciales maestras solo en el gestor del fundador.
- **Variables de entorno** locales en `.env` (nunca commiteadas al repo).
- El developer accede a **logs y métricas**, pero **no a configuración de billing**.

## Por qué importa

Si el developer se va, el fundador conserva 100% del control del producto sin tener que cambiar nada urgente.

---

### Relacionado
- [[Reglas de Seguridad]]
- [[Qué hacer si algo falla]]
- [[Checklist para empezar]]
