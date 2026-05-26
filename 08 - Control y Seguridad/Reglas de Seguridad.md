# Reglas de Seguridad

Mínimos innegociables antes y durante producción.

## En el producto

- **HTTPS obligatorio** en producción.
- **Headers de seguridad** activos (Helmet o equivalente).
- **CORS** configurado solo para los dominios propios.
- **Rate limiting**:
  - 100 requests por minuto por usuario autenticado.
  - 30 requests por minuto por IP sin autenticar.
- **Validación de inputs** en todos los endpoints.
- **Sanitización** para prevenir XSS en chat y otros campos abiertos.
- **Hashing de contraseñas** con bcrypt y salt rounds altos.
- **Tokens de sesión** con expiración corta + refresh con rotación.
- **Webhooks verificados** con firmas (no aceptar nada sin firma válida).

## En el código

- **Nunca** secrets en el repo. Todo en variables de entorno.
- **Nunca** loguear datos sensibles (passwords, tokens, números de tarjeta).
- **Nunca** queries SQL concatenadas con strings (usar el ORM siempre).

## Backups y disaster recovery

- **Base de datos:** backup automático diario.
- **Almacenamiento de archivos:** versionado activado.
- **Código:** GitHub es el backup.
- **Variables de entorno:** documentadas en el gestor de contraseñas.
- **Documentación de recuperación:** instrucciones paso a paso para restaurar desde cero.

---

### Relacionado
- [[Propiedad y Accesos]]
- [[Qué hacer si algo falla]]
