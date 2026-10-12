# Política de seguridad

## Versiones con soporte

| Versión | Soporte de seguridad |
|---------|----------------------|
| Última versión publicada | Sí |
| 1.0.x | Sí |
| Anteriores a 1.0 | No |

Si usas una versión sin soporte, actualiza a la más reciente para recibir las correcciones
de seguridad.

## Cómo reportar una vulnerabilidad

Si encuentras una vulnerabilidad en XicoNemi:

1. **No la publiques.** No abras un issue público ni la comentes en un pull request.
2. **Contacta al equipo** por una de estas vías:
   - correo a `xiconemi@gmail.com`;
   - reporte privado desde la pestaña **Security** del repositorio en GitHub, opción
     **Report a vulnerability** (GitHub Security Advisories).
3. **Incluye en el reporte:**
   - descripción clara del problema;
   - versión o commit afectado;
   - impacto potencial;
   - pasos para reproducirlo.

## Qué esperar

- Respuesta inicial en un plazo de 48 horas.
- Seguimiento mientras se prepara la corrección.
- Si lo deseas, mención en los agradecimientos una vez publicada la solución.

## Buenas prácticas para quien contribuye

- Nunca subas el archivo `.env` ni credenciales al repositorio. Ya está incluido en
  `.gitignore`.
- Usa valores de `JWT_SECRET` largos y aleatorios, distintos en cada entorno.
- Protege con `verifyToken` (y `authorizeRole` cuando corresponda) todo endpoint que lea o
  modifique datos de otros usuarios.
- No devuelvas el campo `password` en las respuestas, aunque esté cifrado.

Gracias por ayudar a mantener XicoNemi seguro.
