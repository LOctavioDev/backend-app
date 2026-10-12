# Variables de entorno

[Volver al README](../README.md)

El proyecto lee su configuración desde variables de entorno. En desarrollo se cargan desde
un archivo `.env` en la raíz (con `dotenv`), y en producción se configuran en la
plataforma de despliegue.

Esta tabla se construyó buscando todas las apariciones de `process.env` en el código, por
lo que refleja lo que la aplicación **realmente** usa. El archivo `.env.example` no está
completo: le faltan varias variables y tiene una que el código no usa.

## Tabla completa

### Núcleo (necesarias para arrancar)

| Variable | Ejemplo | Usada en | Descripción |
|----------|---------|----------|-------------|
| `DATABASE_URL` | `postgresql://user:pass@host:5432/db` | `prisma/schema.prisma`, servicios de respaldo | Conexión a PostgreSQL. Debe ser una URL de PostgreSQL, no de MySQL como sugiere `.env.example`. |
| `MONGO_URI` | `mongodb+srv://user:pass@cluster/db` | `src/config/nosqlDataBase.ts` | Conexión a MongoDB para el chat. Si falta, se muestra un aviso y el chat no funciona. No está en `.env.example`. |
| `REDIS_URL` | `mi-redis.upstash.io` | `src/config/redisConfig.ts` | **Nombre del host** de Redis (no una URL `redis://`). La conexión siempre usa TLS. |
| `REDIS_PORT` | `6379` | `src/config/redisConfig.ts` | Puerto de Redis. |
| `REDIS_PASS` | `********` | `src/config/redisConfig.ts` | Contraseña de Redis. |
| `JWT_SECRET` | cadena aleatoria larga | `services/auth.service.ts`, `middleware/verifyToken.ts` | Clave para firmar y verificar los tokens. |
| `PORT` | `11111` | `src/server.ts` | Puerto HTTP. Valor por defecto: `11111`. |

### Correo

| Variable | Ejemplo | Descripción |
|----------|---------|-------------|
| `EMAIL_HOST` | `smtp.gmail.com` | Servidor SMTP. |
| `EMAIL_PORT` | `587` | Puerto SMTP. |
| `EMAIL_USER` | `cuenta@gmail.com` | Usuario SMTP. |
| `EMAIL_PASS` | `********` | Contraseña SMTP. En Gmail debe ser una **contraseña de aplicación**, no la contraseña de la cuenta. |

Si el servidor SMTP no responde, el registro de usuarios hace que el proceso termine. Ver
[observaciones](OBSERVACIONES.md).

### Inicio de sesión con Google

| Variable | Descripción |
|----------|-------------|
| `GOOGLE_CLIENT` | Client ID de OAuth de Google. Se usa como `audience` al verificar el `idToken`. |

`.env.example` incluye `CLIENT_ID_ANDROID`, pero el código no la lee: la variable correcta
es `GOOGLE_CLIENT`.

### Cloudinary (imágenes)

| Variable | Descripción |
|----------|-------------|
| `CLOUD_NAME` | Nombre de la cuenta de Cloudinary. |
| `API_KEY` | API key de Cloudinary. |
| `API_SECRET` | API secret de Cloudinary. |

Los nombres son genéricos (`API_KEY`, `API_SECRET`). Ten cuidado de no confundirlos con
claves de otros servicios.

### Respaldos

| Variable | Descripción |
|----------|-------------|
| `BOT_TOKEN` | Token del bot de Telegram que envía el respaldo. |
| `CHAT_ID` | ID del chat o canal de Telegram que recibe el archivo. |
| `GOOGLE_SERVICE_ACCOUNT_JSON` | Contenido completo del JSON de una cuenta de servicio de Google, en una sola línea. |
| `FOLDER_ID` | ID de la carpeta de Google Drive donde se suben los respaldos. La carpeta debe estar compartida con la cuenta de servicio. |

### Otras

| Variable | Descripción |
|----------|-------------|
| `NODE_ENV` | Solo se lee en `src/config/index.ts`, archivo que no se importa en ningún lado. En la práctica no tiene efecto. |

## Plantilla completa sugerida

Copia esta plantilla en tu `.env` y completa los valores. Las secciones opcionales pueden
quedar vacías si no usas esas funciones.

```dotenv
# ---------- Núcleo ----------
DATABASE_URL="postgresql://<usuario>:<password>@<host>:<puerto>/<base_de_datos>"
MONGO_URI="mongodb+srv://<usuario>:<password>@<cluster>/<base_de_datos>"
REDIS_URL="<host_de_redis>"
REDIS_PORT=6379
REDIS_PASS="<password_de_redis>"
JWT_SECRET="<cadena_aleatoria_larga>"
PORT=11111

# ---------- Correo ----------
EMAIL_HOST="smtp.gmail.com"
EMAIL_PORT=587
EMAIL_USER="<cuenta>@gmail.com"
EMAIL_PASS="<contraseña_de_aplicación>"

# ---------- Google Sign-In ----------
GOOGLE_CLIENT="<client_id>.apps.googleusercontent.com"

# ---------- Cloudinary ----------
CLOUD_NAME="<cloud_name>"
API_KEY="<api_key>"
API_SECRET="<api_secret>"

# ---------- Respaldos ----------
BOT_TOKEN="<token_del_bot_de_telegram>"
CHAT_ID="<id_del_chat>"
GOOGLE_SERVICE_ACCOUNT_JSON='{"type":"service_account", ...}'
FOLDER_ID="<id_de_carpeta_de_drive>"
```

## Cómo se cargan

No existe un único punto donde se cargue el `.env`. Varios módulos llaman a
`dotenv.config()` por su cuenta (`nosqlDataBase.ts`, `backupService.ts`,
`backupController.ts`), y el cliente de Prisma también lee el `.env` al importarse. Como
`server.ts` importa esos módulos antes que las rutas, las variables ya están disponibles
cuando se leen.

Algunos módulos leen variables **en el momento de importarse** (por ejemplo,
`verifyToken.ts` guarda `JWT_SECRET` en una constante). Cambiar una variable con el
servidor en marcha no tiene efecto: hay que reiniciarlo.

## Seguridad

- El archivo `.env` está en `.gitignore`. Nunca lo subas al repositorio.
- Usa valores distintos en desarrollo y en producción.
- Si una clave se filtra, revócala en el servicio correspondiente y genera una nueva.
