# Observaciones técnicas

[Volver al README](../README.md)

Este documento reúne los detalles encontrados al leer el código completo y ejecutar la API
en un entorno local (PostgreSQL 16, MongoDB 7, Redis 7 con TLS, Node.js 20). Cada punto
indica si se **verificó en ejecución** o si se deduce de la **lectura del código**, y
propone una corrección.

El código no se modificó: este documento sirve como lista de trabajo y como material de
estudio sobre errores frecuentes en APIs con Express.

## Resumen

| # | Área | Observación | Impacto | Verificación |
|---|------|-------------|---------|--------------|
| 1 | Tiempo real | Socket.IO no recibe conexiones | Alto | Ejecución |
| 2 | Correo | Un fallo de SMTP termina el proceso | Alto | Ejecución |
| 3 | Chat | Ningún rol puede leer el historial | Alto | Ejecución |
| 4 | Chat | Los mensajes con IDs reales no se guardan | Alto | Ejecución |
| 5 | Facebook | El token generado no contiene los datos del usuario | Alto | Código |
| 6 | Rutas | `subscriptions` y `userHasRoutes` no están registrados | Medio | Ejecución |
| 7 | Reseñas | El borrado no elimina la reseña | Medio | Ejecución |
| 8 | Puntos de interés | `locationId` validado como número | Medio | Código |
| 9 | Itinerarios | `eventId` se envía a Prisma y provoca un error | Medio | Código |
| 10 | Validación | Se valida el cuerpo pero se guarda el original | Medio | Ejecución |
| 11 | Usuarios | `birthday` en milisegundos desborda la columna | Medio | Ejecución |
| 12 | Correo | El enlace de verificación no incluye el token | Medio | Ejecución |
| 13 | Configuración | Redis exige TLS siempre | Medio | Ejecución |
| 14 | Configuración | `.env.example` incompleto | Medio | Código |
| 15 | Datos | `prisma/seed.ts` no compila con el esquema actual | Bajo | Ejecución (`tsc`) |
| 16 | HTTP | `res.json(...).status(...)` ignora el código | Bajo | Código |
| 17 | Prisma | Una instancia de `PrismaClient` por archivo | Bajo | Código |
| 18 | Errores | Dos estilos de respuesta de error | Bajo | Ejecución |
| 19 | Contraseñas | El mensaje de longitud mínima no coincide | Bajo | Código |
| 20 | Redis | La lista negra de tokens crece sin límite | Bajo | Código |
| 21 | Swagger | Documenta 46 de 73 operaciones | Bajo | Ejecución |
| 22 | Código muerto | Archivos y funciones sin uso | Bajo | Código |

---

## 1. Socket.IO no recibe conexiones

**Archivo:** `src/server.ts`

El constructor crea un servidor HTTP y le adjunta Socket.IO:

```ts
this.httpServer = createServer(this.app);
this.io = new SocketServer(this.httpServer, { cors: { origin: '*' } });
```

Pero `listen()` arranca otro servidor:

```ts
this.app.listen(this.app.get('port'), ...)
```

`app.listen()` crea internamente un `http.Server` **nuevo**, distinto de `httpServer`. El
servidor que escucha el puerto no tiene Socket.IO, y el que lo tiene nunca escucha.

**Evidencia:** el handshake de Socket.IO recibe la respuesta 404 de Express:

```text
GET /socket.io/?EIO=4&transport=polling
{"error":true,"message":"Route not found","details":null}
```

**Corrección:**

```ts
listen() {
  this.httpServer.listen(this.app.get('port'), async () => { ... });
}
```

## 2. Un fallo de SMTP termina el proceso

**Archivo:** `src/models/user.model.ts`

```ts
newAccount(data.name, data.email, data.token);   // sin await ni catch
const user = await prisma.users.create({ data });
```

`newAccount` es asíncrona. Al no esperarla ni capturar su error, si el servidor SMTP falla
la promesa queda rechazada sin manejar. Desde Node.js 15, una promesa rechazada sin
manejar **termina el proceso**.

**Evidencia:** con `EMAIL_HOST` apuntando a un puerto cerrado, `POST /auth/sign-up` registró
`Error: connect ECONNREFUSED 127.0.0.1:2525` y el servidor dejó de responder.

**Corrección:** decidir si el correo es obligatorio. Si no lo es, capturar el error:

```ts
newAccount(data.name, data.email, data.token).catch((err) =>
  console.error('No se pudo enviar el correo de bienvenida', err)
);
```

Además, conviene crear el usuario **antes** de enviar el correo.

## 3. Ningún rol puede leer el historial del chat

**Archivo:** `src/routes/chat.routes.ts`

```ts
authorizeRole(['ADMIN', 'COMMON'])
```

Los roles reales son `SuperAdmin`, `BusinessOwner` y `Common` (con esas mayúsculas).
`ADMIN` y `COMMON` no coinciden con ninguno.

**Evidencia:** un usuario `Common` con token válido recibe
`403 You are not authorized to access this route`.

**Corrección:** `authorizeRole(['SuperAdmin', 'BusinessOwner', 'Common'])`, o simplemente
quitar `authorizeRole` y dejar `verifyToken`.

## 4. Los mensajes con IDs reales no se guardan

**Archivo:** `src/models/chat.model.ts`

```ts
senderId: { type: Number, required: true },
receiverId: { type: Number, required: true },
```

Los usuarios tienen IDs UUID (texto). Mongoose intenta convertir el UUID a número, falla y
la validación rechaza el documento.

**Evidencia:** `POST /api/chat/x/y` con `senderId` UUID responde
`500 Internal server error`.

**Corrección:** declarar ambos campos como `String` (también en la interfaz `IMessage`).

Nota relacionada: `POST /api/chat/:senderId/:receiverId` toma los IDs del cuerpo y no de la
URL, así que los parámetros de la ruta no se usan.

## 5. El token de Facebook no contiene los datos del usuario

**Archivo:** `src/controllers/admin/auth.controller.ts`

```ts
const token = generateToken(user.id);
```

`generateToken` espera el **objeto** usuario y lee `user.id`, `user.email` y `user.type`.
Al recibir solo el ID (un texto), los tres campos del token quedan `undefined`. Con ese
token, `authorizeRole` rechaza al usuario y los controladores que usan `req.user.userId`
no saben quién es.

**Corrección:** `generateToken(user)`.

Otra mejora: verificar que el `accessToken` de Facebook pertenezca a la aplicación propia
(endpoint `debug_token` de la Graph API), y no a cualquier aplicación.

## 6. Routers no registrados

**Archivo:** `src/routes/index.ts`

`subscriptions.routes.ts` y `userHasRoutes.routes.ts` existen pero no se registran.

**Evidencia:** `GET /api/subscriptions` responde `404 Route not found`.

**Corrección:**

```ts
import subscriptionRoutes from './subscriptions.routes';
import userHasRoutes from './userHasRoutes.routes';

router.use('/subscriptions', subscriptionRoutes);
router.use('/user-has-routes', userHasRoutes);
```

Antes de registrar `userHasRoutes`, revisa su esquema Zod: declara `userId` y `routeId`
como números, igual que en el punto 8.

## 7. El borrado de reseñas no elimina nada

**Archivo:** `src/models/review.model.ts`

`deleteReview` busca la reseña y termina: nunca llama a `prisma.reviews.delete`.

**Evidencia:** `DELETE /api/reviews/delete/:id` responde `200` y la reseña sigue existiendo
en `GET /api/reviews/:id`.

**Corrección:**

```ts
async deleteReview(id: string) {
  const review = await prisma.reviews.findUnique({ where: { id } });
  if (!review) throw new AppError('La reseña no existe.', 404);
  await prisma.reviews.delete({ where: { id } });
  return { message: 'Reseña eliminada correctamente.' };
}
```

El mismo archivo tiene otro patrón a revisar: varios métodos capturan su propio
`AppError('...', 404)` y lo reemplazan por un `500` genérico.

## 8. `locationId` validado como número

**Archivo:** `src/models/pointOfInterest.model.ts`

```ts
locationId: z.number().int().min(1, 'Campo requerido'),
```

En la base de datos, `locationId` es un UUID (`String`). Si se envía un número, pasa Zod
pero Prisma lo rechaza. Si se envía el UUID correcto, Zod lo rechaza. Así,
`POST /api/points` no puede completarse.

**Corrección:** `locationId: z.string().uuid()`.

## 9. `eventId` en la creación de itinerarios

**Archivo:** `src/models/itinerary.model.ts`

```ts
const itinerary = await prisma.itineraries.create({ data });
if (data.eventId) { ... }
```

`data` incluye `eventId`, que no es una columna de `Itineraries`. Prisma rechaza campos
desconocidos, así que crear un itinerario **con** `eventId` falla antes de llegar a la
vinculación.

**Corrección:** separar el campo antes de crear, como ya se hace en `event.model.ts`:

```ts
const { eventId, ...itineraryData } = data;
const itinerary = await prisma.itineraries.create({ data: itineraryData });
```

## 10. Se valida el cuerpo, pero se guarda el original

**Archivos:** todos los modelos

```ts
userSchema.parse(data);                // valida
await prisma.users.create({ data });   // guarda el objeto original
```

`schema.parse()` devuelve un objeto **limpio**, solo con los campos declarados. Al
descartarlo y guardar `data`, cualquier campo adicional del cuerpo llega a Prisma. Esto
provoca errores (como en el punto 9) y permite modificar columnas que el cliente no debería
controlar.

**Corrección:** usar siempre el resultado de la validación:

```ts
const parsed = userSchema.parse(data);
await prisma.users.create({ data: { ...parsed, password: hashed, token } });
```

Con `.strict()` en el esquema, Zod además **rechaza** los campos desconocidos en lugar de
ignorarlos.

## 11. `birthday` en milisegundos desborda la columna

**Archivos:** `prisma/schema.prisma`, `src/models/user.model.ts`

La columna es `Int` (32 bits, máximo 2 147 483 647). La validación compara con
`Date.now()` y `new Date('1900-01-01').getTime()`, que están en **milisegundos**, lo que
sugiere que el cliente envíe milisegundos. Un valor como `946684800000` pasa la validación
pero no cabe en la columna.

**Evidencia:** el registro con `birthday` en milisegundos responde `500`; en segundos
funciona.

**Corrección:** documentar y validar segundos (`value <= Date.now() / 1000`), o cambiar la
columna a `BigInt` o `DateTime`.

## 12. El enlace de verificación no incluye el token

**Archivo:** `src/services/emailService.ts`

```ts
// const verificationUrl = `${url_veritication}${token}`;
const verificationUrl = url_mobile;   // 'myapp://verification-success'
```

**Evidencia:** el correo recibido en Mailpit tiene un único enlace:
`myapp://verification-success`.

La app móvil recibe el aviso, pero no el token, así que nadie puede llamar a
`GET /users/verify-email/:token`. Las cuentas quedan con `status = false`. Como los inicios
de sesión no comprueban `status`, esto no impide usar la app, pero el campo pierde su
sentido.

**Corrección:** incluir el token en el enlace de la app (`myapp://verify?token=...`) o usar
la URL de la API. Decidir también si las cuentas sin verificar pueden iniciar sesión.

## 13. Redis exige TLS siempre

**Archivo:** `src/config/redisConfig.ts`

```ts
tls: {},
maxRetriesPerRequest: null,
```

Con un Redis sin TLS, la conexión nunca se completa. Como `maxRetriesPerRequest: null`
hace que los comandos esperen indefinidamente, **toda petición con token se queda colgada**
en `verifyToken`.

**Evidencia:** con Redis local sin TLS, `GET /api/events` con token no respondió en 8
segundos y la consola mostró `Redis Client Error ... ETIMEDOUT` en bucle.

**Corrección:** hacer TLS configurable, por ejemplo con una variable `REDIS_TLS=true`, o
aceptar una URL completa (`rediss://` con TLS, `redis://` sin TLS). Mientras tanto,
[INSTALACION.md](INSTALACION.md#redis-con-tls) explica cómo levantar Redis con TLS en local.

## 14. `.env.example` incompleto

| Problema | Detalle |
|----------|---------|
| Base de datos | Muestra una URL `mysql://`, pero el esquema usa PostgreSQL |
| Faltan | `MONGO_URI`, `GOOGLE_CLIENT`, `CLOUD_NAME`, `API_KEY`, `API_SECRET`, `BOT_TOKEN`, `CHAT_ID`, `GOOGLE_SERVICE_ACCOUNT_JSON`, `FOLDER_ID` |
| Sobra | `CLIENT_ID_ANDROID` no se lee en el código |
| Puerto | `PORT=3000`, pero Swagger está configurado para `11111` |

La plantilla correcta está en [VARIABLES_DE_ENTORNO.md](VARIABLES_DE_ENTORNO.md).

## 15. `prisma/seed.ts` no compila

El script usa nombres de un esquema anterior: `prisma.user`, `prisma.event`,
`prisma.pointOfInterest`, `prisma.promotion`, `prisma.establishment`, `prisma.itinerary`,
`prisma.travel`, y valores de enum como `'COMMON'` o `'ADMIN'`.

**Evidencia:**

```text
prisma/seed.ts(12,18): error TS2551: Property 'user' does not exist on type 'PrismaClient'.
Did you mean 'users'?
```

**Corrección:** reescribirlo con los modelos actuales (`users`, `events`,
`pointsOfInterest`, `promotions`, `businesses`, `itineraries`, `routes`), los enums
actuales y fechas en segundos. Las relaciones requieren IDs reales, así que hay que crear
primero los registros padre y usar sus `id`.

## 16. `res.json(...).status(...)` ignora el código

**Archivos:** controladores de eventos, itinerarios, ubicaciones, puntos de interés,
promociones, rutas y autenticación

```ts
res.json(event).status(201);
```

`res.json()` **envía** la respuesta inmediatamente con el código actual (200). El
`.status(201)` posterior ya no tiene efecto. Por eso las creaciones de esos módulos
responden `200` y no `201`.

**Corrección:** invertir el orden: `res.status(201).json(event)`.

## 17. Una instancia de `PrismaClient` por archivo

Hay `new PrismaClient()` en `config/sqlDataBase.ts`, en cada modelo y en dos controladores.
Cada instancia abre su propio grupo de conexiones, lo que puede agotar el límite de
conexiones de bases de datos administradas.

**Corrección:** importar en todos lados la instancia exportada por
`src/config/sqlDataBase.ts`:

```ts
import prisma from '../config/sqlDataBase';
```

## 18. Dos estilos de respuesta de error

Algunos módulos lanzan `AppError` y responden con el código HTTP correcto. Otros devuelven
`200 { message: "No encontrado" }` o atrapan el error y responden `500` por su cuenta, sin
pasar por `errorHandler`. Los clientes no pueden confiar solo en el código HTTP.

**Corrección:** usar `AppError` + `next(error)` en todos los módulos.

## 19. Longitud mínima de contraseña

`changePassword` exige `newPassword.length >= 9`, pero el mensaje dice
`must be at least 8 characters long`. El registro exige 6. Conviene unificar la regla en
un solo esquema de Zod compartido.

## 20. La lista negra de tokens crece sin límite

`SADD tokenBlacklist <token>` agrega cada token a un único conjunto que nunca se limpia,
aunque los tokens expiren al día.

**Corrección:** guardar cada token como clave con expiración:

```ts
await redisClient.set(`bl:${token}`, '1', 'EX', segundosHastaExpirar);
const revocado = (await redisClient.exists(`bl:${token}`)) === 1;
```

## 21. Cobertura de Swagger

Swagger documenta 46 de las 73 operaciones registradas. Faltan imágenes, itinerarios,
puntos de interés, promociones, rutas, planes y chat. Además, al arrancar, la consola
siempre muestra la URL de documentación de **producción**, también en local.

La referencia completa está en [API.md](API.md).

## 22. Código sin uso

| Elemento | Detalle |
|----------|---------|
| `src/config/index.ts` | Carga `dotenv` según `NODE_ENV`, pero no se importa en ningún lado |
| `src/public/` | Imágenes que no se sirven: no hay `express.static` |
| `services/rateLimitService.ts` | Todo el archivo está comentado |
| `UserModel.getUsersByType`, `UserModel.changePassword` | Vacíos o sin `return` |
| `createUser` en `user.controller.ts` | Exportado, pero ninguna ruta lo usa |
| `getImagesInBD` en `image.controller.ts` | Sin ruta |
| `"back-xiconemi": "file:"` en `package.json` | El paquete se declara como dependencia de sí mismo |
| `fs` en `package.json` | Paquete de npm vacío; `fs` ya es un módulo nativo de Node.js |
| `@types/bcrypt`, `@types/nodemailer` | Están en `dependencies`; corresponden a `devDependencies` |
