# Instalación y entorno local

[Volver al README](../README.md)

Esta guía deja el proyecto funcionando en una computadora local, incluidos los servicios
que necesita (PostgreSQL, MongoDB, Redis y un servidor de correo de pruebas). El
procedimiento se probó con Node.js 20.20, npm 10.8 y Docker en Linux.

## Contenido

1. [Requisitos](#1-requisitos)
2. [Clonar e instalar dependencias](#2-clonar-e-instalar-dependencias)
3. [Levantar los servicios con Docker](#3-levantar-los-servicios-con-docker)
4. [Configurar el archivo .env](#4-configurar-el-archivo-env)
5. [Crear las tablas](#5-crear-las-tablas)
6. [Iniciar el servidor](#6-iniciar-el-servidor)
7. [Probar que todo funciona](#7-probar-que-todo-funciona)
8. [Compilar para producción](#8-compilar-para-producción)
9. [Detener y limpiar](#9-detener-y-limpiar)
10. [Problemas frecuentes](#10-problemas-frecuentes)

## 1. Requisitos

| Herramienta | Versión | Para qué |
|-------------|---------|----------|
| <img src="img/logos/nodedotjs.svg" width="16"> Node.js | 20 o superior | Ejecutar la API |
| <img src="img/logos/npm.svg" width="16"> npm | Incluido con Node.js | Instalar dependencias |
| <img src="img/logos/docker.svg" width="16"> Docker | Cualquier versión reciente | Levantar las bases de datos en local (opcional) |
| `openssl` | Cualquiera | Generar el certificado de Redis en local |
| `pg_dump` | Igual o mayor que la versión de PostgreSQL | Solo para los endpoints de respaldo |

En lugar de Docker se pueden usar servicios en la nube: por ejemplo Neon o Supabase para
PostgreSQL, MongoDB Atlas y Upstash para Redis. El proyecto fue pensado para ese tipo de
servicios (ver el apartado sobre Redis más abajo).

## 2. Clonar e instalar dependencias

```bash
git clone https://github.com/LOctavioDev/backend-app.git
cd backend-app
npm install
```

`npm install` ejecuta al final `prisma generate` (script `postinstall`), que crea el
cliente de Prisma tipado a partir de `prisma/schema.prisma`.

Nota: `package-lock.json` está en `.gitignore`, por lo que cada instalación resuelve las
versiones más recientes compatibles con los rangos de `package.json`.

## 3. Levantar los servicios con Docker

Los puertos de esta guía son poco comunes a propósito, para no chocar con servicios que ya
tengas instalados. Puedes cambiarlos.

### PostgreSQL

```bash
docker run -d --name xico_pg \
  -e POSTGRES_USER=xico -e POSTGRES_PASSWORD=xico -e POSTGRES_DB=xiconemi \
  -p 55432:5432 postgres:16-alpine
```

### MongoDB

```bash
docker run -d --name xico_mongo -p 57017:27017 mongo:7
```

### Redis con TLS

El cliente de Redis del proyecto (`src/config/redisConfig.ts`) se conecta **siempre con
TLS** (`tls: {}`), porque en producción se usa un Redis administrado que lo exige. Un Redis
local sin TLS no sirve: la conexión falla en silencio, el cliente la reintenta sin límite y
**toda petición que envíe un token se queda esperando para siempre**.

Para replicarlo en local sin modificar el código, se levanta Redis con un certificado
autofirmado y se le indica a Node.js que confíe en él.

```bash
# Certificado autofirmado para "localhost"
mkdir -p ~/xico-tls && cd ~/xico-tls
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost" \
  -keyout redis.key -out redis.crt
chmod 644 redis.key
cd -

# Redis escuchando solo con TLS
docker run -d --name xico_redis -p 56379:6379 -v ~/xico-tls:/tls:ro redis:7-alpine \
  redis-server --port 0 --tls-port 6379 \
  --tls-cert-file /tls/redis.crt --tls-key-file /tls/redis.key --tls-auth-clients no
```

### Servidor de correo de pruebas (Mailpit)

Al registrar un usuario la API envía un correo. Si el servidor SMTP no responde, **el
proceso de Node.js termina** (ver [observaciones](OBSERVACIONES.md)). Mailpit recibe los
correos sin enviarlos y los muestra en una interfaz web:

```bash
docker run -d --name xico_mail -p 2525:1025 -p 58025:8025 axllent/mailpit
```

Los correos se ven en `http://localhost:58025`.

### Comprobación

```bash
docker ps --filter name=xico_ --format '{{.Names}}  {{.Status}}'
```

## 4. Configurar el archivo .env

```bash
cp .env.example .env
```

Para el entorno local de esta guía, el `.env` mínimo es:

```dotenv
DATABASE_URL="postgresql://xico:xico@localhost:55432/xiconemi"
MONGO_URI="mongodb://localhost:57017/xiconemi"

REDIS_URL="localhost"
REDIS_PORT=56379
REDIS_PASS=""

JWT_SECRET="cambia-esto-por-un-valor-largo-y-aleatorio"

EMAIL_HOST="localhost"
EMAIL_PORT=2525
EMAIL_USER="test"
EMAIL_PASS="test"

PORT=11111
```

Puntos importantes:

- `DATABASE_URL` debe ser de **PostgreSQL**. El `.env.example` muestra una URL de MySQL,
  pero el esquema de Prisma usa `provider = "postgresql"`.
- `REDIS_URL` es el **nombre del host**, no una URL completa (`redis://...`).
- `PORT=11111` coincide con las URL configuradas para Swagger.
- Las variables de Cloudinary, Google, Telegram y Google Drive solo se necesitan para esas
  funciones. La lista completa está en [VARIABLES_DE_ENTORNO.md](VARIABLES_DE_ENTORNO.md).

Para generar un `JWT_SECRET` aleatorio:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

## 5. Crear las tablas

La carpeta `prisma/migrations` no contiene migraciones, así que las tablas se crean
sincronizando el esquema directamente:

```bash
npx prisma db push
```

Salida esperada:

```text
Your database is now in sync with your Prisma schema.
```

Para explorar los datos con una interfaz visual:

```bash
npx prisma studio
```

## 6. Iniciar el servidor

Como Redis usa un certificado autofirmado, hay que indicarle a Node.js que confíe en él con
`NODE_EXTRA_CA_CERTS`. Esto no hace falta con un Redis en la nube con certificado válido.

```bash
NODE_EXTRA_CA_CERTS=~/xico-tls/redis.crt npm run dev
```

Salida esperada (con colores en la terminal):

```text
Docs https://available-karlotta-ethdev11-59ebf81c.koyeb.app/api/docs
Server running on port 11111
Connected to Redis
Connected to Postgres database
Connected to MongoDB Atlas
```

El primer mensaje siempre muestra la URL de producción, aunque se esté en local. La
documentación local está en `http://localhost:11111/api/docs`.

## 7. Probar que todo funciona

```bash
API=http://localhost:11111/api

# Registro (birthday en segundos Unix; ver nota abajo)
curl -X POST $API/auth/sign-up -H "Content-Type: application/json" -d '{
  "name": "Ana", "lastname": "Lopez", "email": "ana@test.com",
  "password": "secreto123", "tel": "5550000000",
  "gender": "Femenino", "birthday": 946684800
}'

# Inicio de sesión de usuario común
curl -X POST $API/auth/sign-in-common -H "Content-Type: application/json" \
  -d '{"email": "ana@test.com", "password": "secreto123"}'

# Ruta protegida (sustituir <TOKEN> por el token recibido)
curl $API/users/profile -H "Authorization: Bearer <TOKEN>"
```

Respuesta del perfil:

```json
{
  "message": "Profile user",
  "user": {
    "userId": "e5ca596a-...",
    "email": "ana@test.com",
    "type": "Common",
    "iat": 1791781496,
    "exp": 1791867896
  }
}
```

Nota sobre `birthday`: la columna es `Int` de 32 bits. Un valor en milisegundos (como
`Date.now()`) excede su capacidad y la API responde `500`. Usa segundos Unix.

## 8. Compilar para producción

```bash
npm run build     # genera dist/ y copia las plantillas de correo
npm start         # ejecuta dist/index.js
```

La versión compilada también lee el `.env` si existe, pero en producción lo habitual es
definir las variables en la plataforma de despliegue (Koyeb, Render, etc.) y no subir
ningún `.env` al servidor.

## 9. Detener y limpiar

```bash
docker rm -f xico_pg xico_mongo xico_redis xico_mail
rm -rf ~/xico-tls
```

## 10. Problemas frecuentes

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| Las peticiones con token no responden nunca | Redis no acepta TLS o el certificado no es de confianza | Usa Redis con TLS y `NODE_EXTRA_CA_CERTS` (paso 3 y 6) |
| `Redis Client Error ... ETIMEDOUT` repetido en consola | Mismo caso anterior | Igual que arriba |
| El servidor se detiene después de un registro | El servidor SMTP no responde | Levanta Mailpit o configura un SMTP válido |
| `500 Internal server error` al registrarse | `birthday` en milisegundos | Envía segundos Unix |
| `MONGO_URI no está definida` | Falta la variable | Agrégala al `.env` |
| `Error connecting to Postgres database` | `DATABASE_URL` incorrecta o de MySQL | Usa una URL `postgresql://` |
| `Route not found` en `/api/subscriptions` | Ese router no está registrado | Ver [observaciones](OBSERVACIONES.md) |
| El cliente de Socket.IO no conecta | El servidor HTTP que escucha no es el de Socket.IO | Ver [observaciones](OBSERVACIONES.md) |
