<p>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="img/logos/express-white.svg">
    <img src="img/logos/express.svg" alt="Express" height="36">
  </picture>&nbsp;&nbsp;
  <img src="img/logos/swagger.svg" alt="Swagger" height="36">
</p>

# Referencia de la API

[Volver al README](../README.md)

Esta referencia describe las **73 operaciones** registradas en `src/routes/index.ts`. Se
generó leyendo el código de rutas, controladores y modelos, y los flujos principales se
probaron contra una instancia local.

Swagger UI (`/api/docs`) documenta 46 de estas operaciones. Este documento cubre todas.

## Contenido

- [Convenciones](#convenciones)
- [Autenticación](#autenticación-apiauth) `/api/auth`
- [Usuarios](#usuarios-apiusers) `/api/users`
- [Negocios](#negocios-apibusinesses) `/api/businesses`
- [Eventos](#eventos-apievents) `/api/events`
- [Promociones](#promociones-apipromotions) `/api/promotions`
- [Ubicaciones](#ubicaciones-apilocations) `/api/locations`
- [Puntos de interés](#puntos-de-interés-apipoints) `/api/points`
- [Rutas turísticas](#rutas-turísticas-apiroutes) `/api/routes`
- [Itinerarios](#itinerarios-apiitineraries) `/api/itineraries`
- [Reseñas](#reseñas-apireviews) `/api/reviews`
- [Planes](#planes-apiplans) `/api/plans`
- [Estadísticas](#estadísticas-apistats) `/api/stats`
- [Imágenes](#imágenes-apiimages) `/api/images`
- [Chat](#chat-apichat) `/api/chat`
- [Respaldos](#respaldos-apibackup) `/api/backup`
- [Routers no registrados](#routers-no-registrados)

## Convenciones

**URL base:** `http://localhost:11111/api` en desarrollo.

**Formato:** las peticiones y respuestas usan JSON (`Content-Type: application/json`),
salvo la carga de imágenes (`multipart/form-data`).

**Columna "Acceso":**

| Valor | Significado |
|-------|-------------|
| Público | No requiere token. |
| Token | Requiere `Authorization: Bearer <token>`, de cualquier rol. |
| `SA` | Solo `SuperAdmin`. |
| `SA`, `BO` | `SuperAdmin` o `BusinessOwner`. |
| `SA`, `BO`, `C` | Cualquiera de los tres roles (equivale a Token). |

**Identificadores:** todos los `:id` son UUID. Un formato inválido devuelve
`400 { "message": "Invalid UUID format" }`.

**Fechas:** enteros en segundos Unix (por ejemplo `1735689600` para el 1 de enero de 2025).

**Errores:** ver [ARQUITECTURA.md, manejo de errores](ARQUITECTURA.md#manejo-de-errores).
Algunos módulos responden `200` con `{ "message": "..." }` cuando algo no se encuentra;
revisa el contenido de la respuesta.

---

## Autenticación `/api/auth`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| POST | `/sign-up` | Público | Registro de usuario |
| POST | `/sign-in` | Público | Inicio de sesión para el panel (rechaza usuarios `Common`) |
| POST | `/sign-in-common` | Público | Inicio de sesión para la app (acepta cualquier rol) |
| POST | `/google-auth` | Público | Inicio de sesión con Google (solo `SuperAdmin` y `BusinessOwner` existentes) |
| POST | `/facebook-auth` | Público | Inicio de sesión con Facebook (crea el usuario si no existe) |
| POST | `/logout` | Token (en encabezado) | Revoca el token actual |
| PATCH | `/change-password/:id` | Público | Cambio de contraseña con la contraseña anterior |
| DELETE | `/delete-user/:email` | Público | Borra definitivamente un usuario por correo |

### POST `/auth/sign-up`

```json
{
  "name": "Ana",
  "lastname": "Lopez",
  "email": "ana@correo.com",
  "password": "secreto123",
  "tel": "5550000000",
  "gender": "Femenino",
  "birthday": 946684800
}
```

| Campo | Regla |
|-------|-------|
| `name`, `lastname`, `tel` | Texto no vacío |
| `email` | Formato de correo; único |
| `password` | Mínimo 6 caracteres |
| `gender` | `Masculino`, `Femenino` u `Otro` |
| `birthday` | Segundos Unix |

Respuesta `201`:

```json
{ "id": "e5ca596a-...", "message": "Usuario creado correctamente, revisa tu correo." }
```

Errores: `400` validación, `409` correo existente. Envía un correo de bienvenida (ver
[SERVICIOS_EXTERNOS.md](SERVICIOS_EXTERNOS.md#correo)).

### POST `/auth/sign-in` y `/auth/sign-in-common`

```json
{ "email": "ana@correo.com", "password": "secreto123" }
```

Respuesta `200`:

```json
{
  "user": { "id": "...", "name": "Ana", "email": "ana@correo.com", "type": "Common", "password": "", "...": "..." },
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

| Caso | Respuesta |
|------|-----------|
| Falta correo o contraseña | `400 Email and password are required` |
| Correo inexistente | `404 User not found` |
| Contraseña incorrecta | `400 Wrong password or email` |
| Usuario `Common` en `/sign-in` | `403 Access denied. You are not authorized.` |

### POST `/auth/google-auth`

```json
{ "tokenId": "<id_token de Google>" }
```

Verifica el token con Google y busca un usuario `SuperAdmin` o `BusinessOwner` con ese
correo. Responde `{ user, token }` o `403` si no existe.

### POST `/auth/facebook-auth`

```json
{ "accessToken": "<access_token de Facebook>" }
```

Consulta la Graph API, crea un usuario `Common` activo si el correo no existe y responde
`{ token }`.

### POST `/auth/logout`

Sin cuerpo. Envía el token en el encabezado `Authorization`. Responde
`200 { "message": "Logout successful" }` y, a partir de ese momento, el token es rechazado
con `401 Token has been revoked`.

### PATCH `/auth/change-password/:id`

```json
{ "oldPassword": "secreto123", "newPassword": "nuevaClave99", "repeatPassword": "nuevaClave99" }
```

Reglas: los tres campos son obligatorios, `newPassword` y `repeatPassword` deben coincidir
y la nueva contraseña debe tener **al menos 9 caracteres** (el mensaje de error dice 8).

### DELETE `/auth/delete-user/:email`

Borra físicamente al usuario y, en cascada, sus negocios, reseñas, itinerarios y demás
registros relacionados. Responde con el registro eliminado o
`{ "message": "User not found" }`.

---

## Usuarios `/api/users`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| GET | `/profile` | Token | Devuelve el contenido del token |
| GET | `/admin` | `SA` | Comprobación de acceso de administrador |
| GET | `/` | `SA` | Lista de usuarios activos, ordenados por tipo |
| GET | `/:id` | Token | Detalle de un usuario |
| POST | `/super-admin` | `SA` | Crear usuario ya activo (sin correo de verificación) |
| PUT | `/:id` | Token | Actualización completa (todos los campos de registro) |
| PATCH | `/:id` | Token | Actualización parcial |
| PATCH | `/change-password/:id` | Token | Igual que `PATCH /:id` (actualización parcial) |
| DELETE | `/delete/:id` | Token | Desactiva al usuario (`status = false`) |
| GET | `/verify-email/:token` | Público | Activa la cuenta asociada al token de verificación |

### GET `/users/profile`

```json
{
  "message": "Profile user",
  "user": { "userId": "...", "email": "ana@correo.com", "type": "Common", "iat": 1791781496, "exp": 1791867896 }
}
```

### PUT y PATCH `/users/:id`

`PUT` exige todos los campos del registro. `PATCH` acepta cualquier subconjunto, pero al
menos uno. Si se envía `password`, se guarda cifrada. Si se cambia `email`, se verifica que
no pertenezca a otro usuario (`409`).

Para cambiar la contraseña comprobando la anterior, usa
[`PATCH /auth/change-password/:id`](#patch-authchange-passwordid).

### DELETE `/users/delete/:id`

No borra el registro: lo marca como inactivo.

```json
{ "message": "User status updated to inactive successfully", "userId": "..." }
```

---

## Negocios `/api/businesses`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| GET | `/public` | Público | Todos los negocios con su calificación promedio; filtro opcional `?category=` |
| GET | `/` | Token | `SuperAdmin`: todos los negocios. Otros roles: solo los propios |
| GET | `/:id` | Token | Detalle |
| POST | `/` | `SA`, `BO` | Crear |
| PUT | `/:id` | `SA`, `BO` | Actualizar (cuerpo completo) |
| DELETE | `/:id` | `SA`, `BO` | Eliminar definitivamente |

### Cuerpo para crear o actualizar

```json
{
  "ownerId": "4b14596b-...",
  "name": "Cafe Centro",
  "description": "Cafetería en el centro",
  "category": "gastronomia",
  "address": "Av. Principal 10",
  "tel": "5550000000",
  "status": true,
  "web_site": "https://cafecentro.mx",
  "social_networks": { "facebook": "https://facebook.com/cafecentro" }
}
```

| Campo | Regla |
|-------|-------|
| `ownerId`, `name`, `description`, `address`, `tel` | Obligatorios |
| `category` | Obligatorio. Valor de `CategoryType`, sin importar mayúsculas |
| `status` | Booleano obligatorio |
| `social_networks` | Opcional. Objeto de texto a texto |
| `name` | Al crear, no puede repetirse |

Respuesta al crear: `201 { "id": "...", "message": "Negocio creado correctamente." }`.

### GET `/businesses/public?category=Gastronomia`

```json
[
  {
    "id": "de45504d-...",
    "name": "Cafe Centro",
    "category": "Gastronomia",
    "...": "...",
    "averageRating": 4.5
  }
]
```

`averageRating` es el promedio de las reseñas del negocio (0 si no tiene). Si no hay
resultados: `{ "message": "No hay negocios disponibles." }`. A diferencia de `POST`, aquí
la categoría debe escribirse exactamente como en el enum.

---

## Eventos `/api/events`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| GET | `/` | Token | Lista de eventos |
| GET | `/:id` | Token | Detalle |
| GET | `/business/:id` | Público | Eventos de un negocio (registros de `businessHasEvent` con el evento incluido) |
| POST | `/` | Token | Crear y vincular a un negocio |
| PUT | `/:id` | `SA`, `BO` | Actualizar |
| DELETE | `/delete/:id` | `SA`, `BO` | Eliminar |

### POST `/events`

```json
{
  "businessId": "de45504d-...",
  "userId": "4b14596b-...",
  "name": "Festival del café",
  "description": "Degustación y música en vivo",
  "startDate": 1735689600,
  "endDate": 1735776000,
  "type": "Gastronomico"
}
```

`businessId` debe existir. El evento se crea y se vincula al negocio en `businessHasEvent`.
`type` no está en la validación de Zod, pero el esquema de la base de datos lo exige.

---

## Promociones `/api/promotions`

| Método | Ruta | Acceso |
|--------|------|--------|
| GET | `/` | Token |
| GET | `/:id` | Token |
| POST | `/` | Token |
| PUT | `/:id` | Token |
| DELETE | `/delete/:id` | Token |

```json
{ "businessId": "de45504d-...", "name": "2x1 en café", "description": "Lunes a viernes", "status": true }
```

---

## Ubicaciones `/api/locations`

| Método | Ruta | Acceso |
|--------|------|--------|
| GET | `/` | Token |
| GET | `/:id` | Token |
| POST | `/` | Token |
| PUT | `/:id` | Token |
| DELETE | `/delete/:id` | Token |

```json
{ "street": "Calle 5 de Mayo", "lat": "20.2770", "lng": "-97.9580" }
```

`lat` y `lng` se envían como texto; se guardan como `Decimal`.

---

## Puntos de interés `/api/points`

| Método | Ruta | Acceso |
|--------|------|--------|
| GET | `/` | Token |
| GET | `/:id` | Token |
| POST | `/` | Token |
| PUT | `/:id` | Token |
| DELETE | `/delete/:id` | Token |

```json
{
  "name": "Mirador",
  "description": "Vista panorámica",
  "url_image": "https://...",
  "locationId": "<uuid de Location>",
  "status": true
}
```

La validación de Zod declara `locationId` como número, mientras que la base de datos espera
un UUID (texto). Ver [observaciones](OBSERVACIONES.md).

---

## Rutas turísticas `/api/routes`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| GET | `/` | Público | Lista |
| GET | `/:id` | `SA`, `BO`, `C` | Detalle |
| POST | `/` | `SA`, `BO` | Crear |

```json
{
  "ownerId": "4b14596b-...",
  "name": "Sendero al mirador",
  "description": "Recorrido de 5 km",
  "distance": 5.2,
  "time": 7200,
  "status": true,
  "stravaData": { "activityId": 123 }
}
```

`distance` y `time` deben ser positivos. `stravaData` acepta cualquier JSON.

---

## Itinerarios `/api/itineraries`

| Método | Ruta | Acceso |
|--------|------|--------|
| GET | `/` | Token |
| GET | `/:id` | Token |
| POST | `/` | Token |
| PUT | `/:id` | Token |
| DELETE | `/delete/:id` | Token |

```json
{
  "userId": "4b14596b-...",
  "name": "Fin de semana en Xicotepec",
  "startDate": 1735689600,
  "endDate": 1735862400,
  "status": true
}
```

El modelo acepta un `eventId` opcional para vincular el itinerario a un evento. Ver
[observaciones](OBSERVACIONES.md) sobre este campo.

---

## Reseñas `/api/reviews`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| GET | `/` | Público | Todas las reseñas |
| GET | `/:id` | Público | Detalle |
| GET | `/business/:businessId` | Público | Reseñas de un negocio (`404` si no hay) |
| POST | `/` | Público | Crear |
| PUT | `/:id` | Token | Actualizar |
| DELETE | `/delete/:id` | Token | Eliminar (ver observaciones) |

```json
{ "userId": "4b14596b-...", "businessId": "de45504d-...", "rating": 4.5, "comment": "Excelente café" }
```

| Campo | Regla |
|-------|-------|
| `userId` | Obligatorio |
| `businessId` o `eventId` | Opcionales; normalmente se envía uno |
| `rating` | Número mayor o igual a 1; se guarda con un decimal |
| `comment` | Texto no vacío |

---

## Planes `/api/plans`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| GET | `/` | Público | Lista (`404` si no hay planes) |
| GET | `/:id` | Público | Detalle |
| POST | `/` | Token | Crear |
| PUT | `/:id` | Token | Actualizar |
| DELETE | `/delete/:id` | Token | Eliminar; responde `204` sin cuerpo |

```json
{ "name": "Premium", "price": 199.0, "features": "Destacado en búsquedas, estadísticas", "status": true }
```

---

## Estadísticas `/api/stats`

| Método | Ruta | Acceso | Respuesta |
|--------|------|--------|-----------|
| GET | `/user-gender` | Token | `[{ "gender": "Femenino", "count": 1, "percentage": 100 }]` |
| GET | `/average-rating` | Token | `[{ "businessId": "...", "name": "Cafe Centro", "rating": "4.5" }]` |
| GET | `/total-clients-by-plan` | Token | `[{ "planId": "...", "planName": "Premium", "totalClients": 3 }]` |

Pensados para las gráficas del panel de administración.

---

## Imágenes `/api/images`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| PUT | `/:tableName/:id` | Público | Sube una imagen a Cloudinary y guarda la URL en `url_image` del registro |
| GET | `/business` | Público | `id`, `name` y `url_image` de todos los negocios |

`tableName` admite: `Users`, `Businesses`, `Events`, `pointsOfInterest`, `Routes`.

```bash
curl -X PUT http://localhost:11111/api/images/Businesses/<id> \
  -F "image=@foto.jpg"
```

La imagen se recibe en memoria con `multer` (campo `image`) y se sube a la carpeta
`profile` de Cloudinary. Respuesta: `{ "message": "Image uploaded successfully", "data": {...} }`.

---

## Chat `/api/chat`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| GET | `/:userId/:receiverId` | Token + roles `ADMIN`, `COMMON` | Historial entre dos usuarios, ordenado por fecha |
| POST | `/:senderId/:receiverId` | Público | Guarda un mensaje |

```json
{ "senderId": "...", "receiverId": "...", "message": "Hola" }
```

El endpoint de envío lee los IDs del **cuerpo**, no de la URL. Ver
[observaciones](OBSERVACIONES.md) sobre los roles y el tipo de los IDs. El chat en tiempo
real se describe en [ARQUITECTURA.md](ARQUITECTURA.md#tiempo-real-con-socketio).

---

## Respaldos `/api/backup`

| Método | Ruta | Acceso | Descripción |
|--------|------|--------|-------------|
| POST | `/pg` | `SA` | Genera un respaldo con `pg_dump` y lo envía por Telegram |
| POST | `/drive` | `SA` | Genera un respaldo con `pg_dump` y lo sube a Google Drive |

Requieren `pg_dump` instalado en el servidor. Detalles en
[SERVICIOS_EXTERNOS.md](SERVICIOS_EXTERNOS.md#respaldos).

---

## Routers no registrados

Estos archivos existen, pero no se registran en `src/routes/index.ts`, así que sus rutas
responden `404 Route not found`:

| Archivo | Prefijo previsto | Operaciones |
|---------|------------------|-------------|
| `src/routes/subscriptions.routes.ts` | `/api/subscriptions` | CRUD de suscripciones (5) |
| `src/routes/userHasRoutes.routes.ts` | `/api/user-has-routes` (sugerido) | CRUD de rutas seguidas por usuarios (5) |

Para activarlos basta con agregar en `src/routes/index.ts`:

```ts
import subscriptionRoutes from './subscriptions.routes';
router.use('/subscriptions', subscriptionRoutes);
```
