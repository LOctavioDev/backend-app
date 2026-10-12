<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../docs/img/logos/prisma-white.svg">
  <img src="../docs/img/logos/prisma.svg" alt="Prisma" height="40">
</picture>

# Comandos de Prisma

[Volver al README](../README.md) | [Base de datos](../docs/BASE_DE_DATOS.md)

Referencia rápida del CLI de [Prisma](https://www.prisma.io/docs/orm/reference/prisma-cli-reference).
Todos los comandos se ejecutan desde la raíz del proyecto y leen `DATABASE_URL` del `.env`.

## Uso habitual en este proyecto

| Situación | Comando |
|-----------|---------|
| Después de `git pull` o de cambiar `schema.prisma` | `npx prisma generate` |
| Crear o actualizar las tablas en desarrollo | `npx prisma db push` |
| Ver y editar datos en el navegador | `npx prisma studio` |

`npm install` ya ejecuta `prisma generate` automáticamente (script `postinstall`).

## Cliente

### Generar el cliente

```bash
npx prisma generate
```

Crea el cliente tipado (`@prisma/client`) a partir de `schema.prisma`. Hay que ejecutarlo
después de cada cambio en el esquema; si no, TypeScript no reconocerá los modelos o campos
nuevos.

## Sincronizar el esquema

### Sin migraciones: `db push`

```bash
npx prisma db push
```

Aplica el esquema directamente a la base de datos, sin crear archivos de migración. Es la
forma en que se trabaja actualmente en este proyecto, porque `prisma/migrations` no
contiene migraciones. Si un cambio implica perder datos (por ejemplo, borrar una columna),
Prisma pide confirmación.

### Con migraciones: `migrate`

Las migraciones guardan cada cambio como un archivo SQL versionado, lo que permite
reproducir la evolución del esquema en todos los entornos.

```bash
# Crear y aplicar una migración en desarrollo
npx prisma migrate dev --name nombre_descriptivo

# Aplicar en producción las migraciones pendientes
npx prisma migrate deploy

# Ver qué migraciones están aplicadas
npx prisma migrate status
```

Para pasar de `db push` a migraciones en una base que ya tiene tablas, consulta la guía
oficial de [baselining](https://www.prisma.io/docs/orm/prisma-migrate/workflows/baselining).

### Reiniciar la base de datos

```bash
npx prisma migrate reset
```

**Borra todos los datos**, vuelve a aplicar las migraciones y ejecuta el seed. Úsalo solo en
desarrollo.

## Inspección

### Traer el esquema desde la base de datos

```bash
npx prisma db pull
```

Lee la estructura de una base existente y **sobrescribe** `schema.prisma` para reflejarla.
No modifica datos. Útil cuando la base se cambió por fuera de Prisma.

### Prisma Studio

```bash
npx prisma studio
```

Abre una interfaz web (por defecto en `http://localhost:5555`) para ver y editar registros.

### Validar y formatear el esquema

```bash
npx prisma validate   # comprueba que schema.prisma no tenga errores
npx prisma format     # ordena y alinea el archivo
```

## Otros comandos

### Ejecutar SQL directamente

```bash
npx prisma db execute --file script.sql --schema prisma/schema.prisma
```

Ejecuta un archivo SQL contra la base definida en el esquema (es decir, en `DATABASE_URL`).
La opción `--schema` (o `--url`) es obligatoria.

### Poblar con datos de ejemplo

```bash
npx prisma db seed
```

Ejecuta el script de seed configurado en la sección `"prisma": { "seed": ... }` de
`package.json`. Este proyecto no tiene esa sección; usa en su lugar `npm run seed`, que
ejecuta `prisma/seed.ts`. Ese archivo está desactualizado respecto al esquema actual (ver
[observaciones](../docs/OBSERVACIONES.md#15-prismaseedts-no-compila)).
