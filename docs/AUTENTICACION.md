<p>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="img/logos/jsonwebtokens-white.svg">
    <img src="img/logos/jsonwebtokens.svg" alt="JSON Web Tokens" height="36">
  </picture>&nbsp;&nbsp;
  <img src="img/logos/redis.svg" alt="Redis" height="36">&nbsp;&nbsp;
  <img src="img/logos/google.svg" alt="Google" height="36">&nbsp;&nbsp;
  <img src="img/logos/facebook.svg" alt="Facebook" height="36">
</p>

# Autenticación y autorización

[Volver al README](../README.md)

## Contenido

- [Conceptos](#conceptos)
- [Roles](#roles)
- [El token JWT](#el-token-jwt)
- [Registro y verificación de correo](#registro-y-verificación-de-correo)
- [Inicio de sesión con correo y contraseña](#inicio-de-sesión-con-correo-y-contraseña)
- [Inicio de sesión con Google](#inicio-de-sesión-con-google)
- [Inicio de sesión con Facebook](#inicio-de-sesión-con-facebook)
- [Protección de rutas](#protección-de-rutas)
- [Cierre de sesión con lista negra en Redis](#cierre-de-sesión-con-lista-negra-en-redis)
- [Contraseñas](#contraseñas)

## Conceptos

- **Autenticación:** comprobar **quién** hace la petición. Aquí se resuelve con un token
  JWT que el cliente envía en cada petición.
- **Autorización:** decidir **qué puede hacer** quien hace la petición. Aquí se resuelve
  con el rol (`type`) guardado dentro del token.

## Roles

Definidos en el enum `TypeUser` del esquema de Prisma:

| Rol | Pensado para | Inicio de sesión |
|-----|--------------|------------------|
| `Common` | Turistas que usan la app móvil | `/auth/sign-in-common`, Facebook |
| `BusinessOwner` | Dueños de negocios | `/auth/sign-in`, `/auth/sign-in-common`, Google |
| `SuperAdmin` | Administradores de la plataforma | `/auth/sign-in`, `/auth/sign-in-common`, Google |

Todo usuario nuevo se crea como `Common` (valor por defecto del esquema). Un `SuperAdmin`
puede crear usuarios con otro rol desde `POST /api/users/super-admin`.

## El token JWT

Un JWT (JSON Web Token) es una cadena firmada con tres partes separadas por puntos:
encabezado, contenido (payload) y firma. El servidor no guarda sesiones: le basta con
verificar la firma para confiar en el contenido.

`src/services/auth.service.ts` lo genera así:

```ts
const payload = { userId: user.id, email: user.email, type: user.type };
jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: '1d' });
```

| Propiedad | Valor |
|-----------|-------|
| Algoritmo | HS256 (por defecto de `jsonwebtoken`) |
| Clave | `JWT_SECRET` |
| Vigencia | 1 día |
| Contenido | `userId`, `email`, `type`, `iat` (emitido), `exp` (expira) |

El contenido de un JWT **no está cifrado**, solo firmado: cualquiera puede leerlo (por
ejemplo en [jwt.io](https://jwt.io/)). Por eso no se guardan datos sensibles en él.

El cliente debe enviarlo en cada petición protegida:

```http
GET /api/users/profile HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## Registro y verificación de correo

```mermaid
sequenceDiagram
    participant App
    participant API
    participant DB as PostgreSQL
    participant SMTP
    App->>API: POST /auth/sign-up
    API->>API: Validar con Zod
    API->>DB: ¿Existe el correo?
    API->>API: bcrypt.hash(password), generar token de verificación
    API-)SMTP: Enviar correo "Verify your email" (sin esperar)
    API->>DB: INSERT Users (status = false)
    API-->>App: 201 { id, message }
    Note over App,API: Más tarde
    App->>API: GET /users/verify-email/:token
    API->>DB: Buscar usuario con ese token
    API->>DB: status = true, token = null
    API-->>App: { message: "Cuenta activada correctamente" }
```

Detalles:

- El token de verificación es una cadena aleatoria generada en `src/utils/token.ts` con
  `Math.random()` y la fecha actual.
- El correo se envía con la plantilla `src/services/emails/templates/newAccount.html`. En
  la versión actual, el botón del correo apunta a `myapp://verification-success` (un enlace
  de la app móvil) y no incluye el token; el enlace de verificación por API está comentado
  en `emailService.ts`.
- Los inicios de sesión no comprueban `status`, así que una cuenta sin verificar puede
  iniciar sesión.

## Inicio de sesión con correo y contraseña

```mermaid
flowchart TD
    A[POST /auth/sign-in o /sign-in-common] --> B{¿email y password?}
    B -- No --> E1[400 Email and password are required]
    B -- Sí --> C{¿Existe el usuario?}
    C -- No --> E2[404 User not found]
    C -- Sí --> D{¿Ruta /sign-in y rol Common?}
    D -- Sí --> E3[403 Access denied]
    D -- No --> F{bcrypt.compare}
    F -- No coincide --> E4[400 Wrong password or email]
    F -- Coincide --> G[generateToken]
    G --> H["200 { user, token }<br/>(password vaciado)"]
```

La diferencia entre las dos rutas es solo la regla del rol: `/sign-in` está pensada para
el panel de administración y bloquea a los usuarios `Common`.

## Inicio de sesión con Google

```mermaid
sequenceDiagram
    participant App
    participant Google
    participant API
    participant DB as PostgreSQL
    App->>Google: Inicio de sesión (SDK de Google)
    Google-->>App: idToken
    App->>API: POST /auth/google-auth { tokenId }
    API->>Google: verifyIdToken(tokenId, audience = GOOGLE_CLIENT)
    Google-->>API: payload con email
    API->>DB: Buscar usuario SuperAdmin o BusinessOwner con ese email
    alt existe
        API-->>App: 200 { user, token }
    else no existe
        API-->>App: 403 Access denied
    end
```

Solo permite entrar a cuentas de administración que ya existen; no crea usuarios.

## Inicio de sesión con Facebook

```mermaid
sequenceDiagram
    participant App
    participant FB as Facebook Graph API
    participant API
    participant DB as PostgreSQL
    App->>API: POST /auth/facebook-auth { accessToken }
    API->>FB: GET /me?fields=id,name,email
    FB-->>API: { id, name, email }
    API->>DB: ¿Existe el email?
    alt no existe
        API->>DB: Crear usuario Common activo (password vacío)
    else existe
        API->>DB: status = true
    end
    API-->>App: 200 { token }
```

Ver [observaciones](OBSERVACIONES.md) sobre el contenido del token que genera esta ruta.

## Protección de rutas

Dos middlewares en `src/middleware/` se encadenan en la definición de cada ruta:

```ts
router.post('/', verifyToken, authorizeRole(['SuperAdmin', 'BusinessOwner']), createBusiness);
```

### `verifyToken`

1. Lee el encabezado `Authorization` y extrae el token después de `Bearer `.
2. Si no hay token: `401 No token provided`.
3. Consulta en Redis si el token está revocado: `401 Token has been revoked`.
4. Verifica firma y vigencia con `jwt.verify`: si falla, `401 Invalid token`.
5. Guarda el contenido en `req.user` y continúa.

Para que TypeScript conozca `req.user`, el archivo amplía la interfaz `Request` de Express
con `declare global`.

### `authorizeRole(roles)`

Es una función que **devuelve** un middleware (patrón "fábrica"). Comprueba que
`req.user.type` esté en la lista permitida; si no, responde
`403 You are not authorized to access this route`.

```ts
export const authorizeRole = (allowedRoles: string[]) => {
  return (req, res, next) => {
    if (!req.user || !allowedRoles.includes(req.user.type)) {
      return next(new AppError('You are not authorized to access this route', 403));
    }
    next();
  };
};
```

Los nombres de rol deben coincidir exactamente con el enum (`SuperAdmin`,
`BusinessOwner`, `Common`).

## Cierre de sesión con lista negra en Redis

Un JWT es válido hasta que expira; el servidor no puede "borrarlo". Para cerrar sesión
antes de tiempo, el proyecto mantiene una **lista negra**:

```mermaid
sequenceDiagram
    participant App
    participant API
    participant R as Redis
    App->>API: POST /auth/logout (Bearer T)
    API->>R: SADD tokenBlacklist T
    API-->>App: 200 Logout successful
    App->>API: GET /users/profile (Bearer T)
    API->>R: SISMEMBER tokenBlacklist T
    R-->>API: 1
    API-->>App: 401 Token has been revoked
```

Implementación en `src/middleware/tokenBlacklist.ts`:

| Función | Comando de Redis |
|---------|------------------|
| `addToBlacklist(token)` | `SADD tokenBlacklist <token>` |
| `isTokenBlacklisted(token)` | `SISMEMBER tokenBlacklist <token>` |

El conjunto no tiene caducidad, así que crece con cada cierre de sesión. Una mejora habitual
es guardar cada token como clave individual con `SET token 1 EX <segundos_restantes>`, para
que Redis lo elimine solo cuando el JWT ya habría expirado.

## Contraseñas

`src/utils/bcrypt.ts` usa **bcrypt** con 10 rondas de sal:

- `hashPassword(password)`: genera el hash que se guarda en la base de datos.
- `comparePassword(password, hash)`: compara la contraseña recibida con el hash guardado.

bcrypt es lento a propósito, lo que encarece los ataques de fuerza bruta, e incluye una sal
aleatoria en cada hash, así que dos contraseñas iguales producen hashes distintos.
