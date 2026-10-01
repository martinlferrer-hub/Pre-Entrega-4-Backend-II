# EventHub API

API REST para una plataforma de eventos e inscripciones.

## Pre-entrega 4 — Autenticación centralizada con Passport.js

En esta etapa se refactoriza la autenticación implementada en la Pre-entrega 3 para centralizar el registro, login y usuario actual mediante estrategias de **Passport.js**.

El contrato externo de la API se mantiene sin cambios: las rutas, métodos y respuestas principales siguen siendo los mismos. Lo que cambia es la organización interna de la autenticación.

Se mantienen:

- Registro seguro con `bcrypt`.
- Login con credenciales.
- JWT firmado con `JWT_SECRET`.
- Cookie HTTP Only `currentUser`.
- Ruta protegida `/current`.
- Logout mediante eliminación de la cookie.

Passport queda además preparado para incorporar futuros providers externos, como Google o GitHub, sin modificar `app.js`.

## Tecnologías

- Node.js
- Express
- MongoDB
- Mongoose
- Passport.js
- passport-local
- passport-custom
- bcrypt
- jsonwebtoken
- cookie-parser
- dotenv
- JavaScript / ECMAScript Modules (ESM)

## Instalación

```bash
npm install
```

Crear un archivo `.env` a partir de `.env.example`.

```env
PORT=8080
NODE_ENV=development
MONGO_URL=mongodb://localhost:27017/eventhub
JWT_SECRET=change_this_secret
JWT_EXPIRES_IN=1h
```

`JWT_SECRET` y `MONGO_URL` son obligatorios. No subir `.env` a GitHub.

## Ejecución

Modo desarrollo:

```bash
npm run dev
```

Modo normal:

```bash
npm start
```

Tests básicos:

```bash
npm test
```

## Variables de entorno

| Variable | Descripción |
|---|---|
| `PORT` | Puerto del servidor Express |
| `NODE_ENV` | Entorno de ejecución (`development` / `production`) |
| `MONGO_URL` | URL de conexión a MongoDB |
| `JWT_SECRET` | Clave obligatoria utilizada para firmar y verificar JWT |
| `JWT_EXPIRES_IN` | Tiempo de expiración del JWT, por ejemplo `1h` |

## Arquitectura de autenticación

Passport se inicializa en `src/app.js`, mientras que todas las estrategias están centralizadas en `src/config/passport.config.js`.

```text
Route
  ↓
Passport Strategy
  ↓
Repository / DAO / Utils
  ↓
Model / MongoDB
```

### Estrategia `register`

La estrategia `register` se encarga de:

- Validar campos obligatorios.
- Validar tipos.
- Normalizar nombre, apellido y email.
- Validar formato del email.
- Validar longitud mínima de contraseña.
- Comprobar email duplicado.
- Hashear la contraseña mediante `bcrypt`.
- Crear el usuario con `role: "user"`.
- Ignorar cualquier `role` enviado desde el registro público.

La contraseña nunca se devuelve.

### Estrategia `login`

La estrategia `login`:

- Valida la presencia de email y contraseña.
- Normaliza el email.
- Busca el usuario.
- Compara la contraseña con `bcrypt.compare()`.
- Devuelve un usuario seguro para el controller.
- Utiliza el mensaje genérico `Credenciales inválidas` ante cualquier fallo de autenticación.

La estrategia **no genera el JWT**.

Después de una autenticación exitosa, `sessions.controller.js` genera el JWT y establece la cookie `currentUser`.

### Estrategia `current`

La estrategia `current`:

- Lee la cookie `currentUser`.
- Verifica el JWT.
- Deja el payload seguro disponible en `req.user`.
- Rechaza la solicitud si no existe un token válido.

### Preparación para providers externos

Las estrategias están centralizadas en `src/config/passport.config.js`. Esta organización permite agregar futuras estrategias como Google o GitHub sin colocar lógica de autenticación en `app.js`.

## Estructura del proyecto

```text
src/
├── app.js
├── server.js
├── config/
│   ├── database.js
│   ├── env.js
│   └── passport.config.js
├── routes/
│   ├── events.router.js
│   ├── health.router.js
│   └── sessions.router.js
├── controllers/
│   ├── events.controller.js
│   ├── health.controller.js
│   └── sessions.controller.js
├── services/
│   ├── events.service.js
│   └── sessions.service.js
├── repositories/
│   ├── events.repository.js
│   └── users.repository.js
├── dao/
│   ├── events.dao.js
│   └── users.dao.js
├── models/
│   ├── Event.js
│   └── User.js
├── middlewares/
│   └── error.middleware.js
└── utils/
    ├── hash.js
    └── jwt.js
```

## Rutas disponibles

### GET `/api/health`

Verifica que el servidor esté activo.

Response `200`:

```json
{
  "status": "ok",
  "message": "Servidor activo"
}
```

### GET `/api/events`

Endpoint base de eventos.

Response `200`:

```json
{
  "status": "success",
  "payload": []
}
```

### GET `/api/sessions`

Endpoint base de sessions mantenido desde la primera entrega.

Response `200`:

```json
{
  "status": "success",
  "payload": []
}
```

### POST `/api/sessions/register`

Registro público de usuarios mediante Passport.

Request:

```json
{
  "first_name": "Ana",
  "last_name": "Pérez",
  "email": "Ana@Mail.com ",
  "password": "Secreta123"
}
```

Response `201`:

```json
{
  "status": "success",
  "payload": {
    "id": "665f2a...",
    "first_name": "Ana",
    "last_name": "Pérez",
    "email": "ana@mail.com",
    "role": "user"
  }
}
```

La contraseña no se devuelve y el `role` enviado por el cliente no se utiliza para el registro público.

Email duplicado:

```json
{
  "status": "error",
  "message": "El email ya está registrado"
}
```

### POST `/api/sessions/login`

Autenticación mediante la estrategia Passport `login`.

Request:

```json
{
  "email": "ana@mail.com",
  "password": "Secreta123"
}
```

Response `200`:

```json
{
  "status": "success",
  "message": "Login correcto"
}
```

Además, el controller genera el JWT y establece la cookie `currentUser` con:

- `httpOnly: true`
- `sameSite: "lax"`
- `maxAge: 3600000`
- `secure: false` en desarrollo
- `secure: true` en producción

El JWT contiene únicamente:

```json
{
  "id": "665f2a...",
  "email": "ana@mail.com",
  "role": "user"
}
```

Credenciales inválidas:

```json
{
  "status": "error",
  "message": "Credenciales inválidas"
}
```

El mensaje es genérico para no revelar si el email existe o si la contraseña fue incorrecta.

### GET `/api/sessions/current`

Ruta protegida mediante la estrategia Passport `current`.

Debe enviarse la cookie `currentUser` obtenida durante el login.

Response `200`:

```json
{
  "status": "success",
  "payload": {
    "id": "665f2a...",
    "email": "ana@mail.com",
    "role": "user"
  }
}
```

Sin cookie, con un token inválido o con un token expirado:

Response `401`:

```json
{
  "status": "error",
  "message": "No autenticado"
}
```

### POST `/api/sessions/logout`

No requiere Passport. Elimina la cookie `currentUser`.

Response `200`:

```json
{
  "status": "success",
  "message": "Sesión cerrada"
}
```

## Casos de prueba recomendados

1. Registro exitoso.
2. Registro con email duplicado.
3. Login correcto.
4. Login con email inexistente.
5. Login con contraseña incorrecta.
6. `/current` con cookie válida.
7. `/current` sin cookie.
8. `/current` con token manipulado.
9. Logout y posterior acceso a `/current`.

Flujo completo:

```text
REGISTER
   ↓
LOGIN
   ↓
Cookie currentUser
   ↓
CURRENT → 200
   ↓
LOGOUT
   ↓
CURRENT → 401
```

## Seguridad

- Las contraseñas se almacenan utilizando `bcrypt`.
- El modelo User utiliza `select: false` para `password`.
- El JWT no contiene la contraseña.
- Las respuestas de autenticación no contienen la contraseña.
- La cookie de autenticación utiliza `httpOnly`.
- `secure` se activa en producción.
- `JWT_SECRET` es obligatorio y no está hardcodeado.
- `.env` y credenciales privadas no deben subirse al repositorio.

## Autor

Martin Ferrer
