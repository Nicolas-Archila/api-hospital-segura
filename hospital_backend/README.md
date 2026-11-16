# API Hospital — Backend

API REST en **Node.js + Express 5 (CommonJS)** conectada a **PostgreSQL en Supabase** mediante `pg` con TLS verificado. Documentada con Swagger/OpenAPI.

## 1. Requisitos

- Node.js 20 o superior y npm.
- Base de datos creada según `database/README.md`.
- Certificado CA de Supabase en `certs/prod-ca-2021.crt` (Supabase → Database → Settings → SSL configuration → Download certificate).

## 2. Configuración

Copia el ejemplo y edítalo:

```powershell
copy .env.example .env
```

| Variable | Descripción | Ejemplo |
|---|---|---|
| `PORT` | Puerto de la API | `3000` |
| `ALLOWED_ORIGIN` | Orígenes permitidos por CORS, separados por coma, sin espacios | `http://localhost:5173,http://localhost:3000` |
| `DATABASE_URL` | Cadena de conexión de Supabase (Session pooler, puerto 5432). Codifica los caracteres especiales de la contraseña | `postgresql://USUARIO:CONTRASENA@HOST:5432/postgres` |
| `JWT_SECRET` | Secreto para firmar los JWT. Mínimo 64 caracteres aleatorios | — |
| `JWT_EXPIRES_IN` | Duración de los tokens | `1h` |
| `API_KEY_POSTMAN`, `API_KEY_ADMIN`, `API_KEY_MOVIL` | API Keys para integraciones privadas | — |
| `ADMIN_NOMBRE`, `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Solo para crear el primer administrador. Retirar después | — |

Para generar secretos aleatorios:

```powershell
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

**Incluye los dos orígenes en `ALLOWED_ORIGIN`:** `5173` para el frontend y `3000` para que Swagger UI pueda enviar peticiones POST.

## 3. Instalación y arranque

```powershell
npm ci
node scripts/probar-conexion.js
npm run dev
```

| Comando | Para qué sirve |
|---|---|
| `npm ci` | Instala las dependencias exactas de `package-lock.json` |
| `node scripts/probar-conexion.js` | Comprueba la conexión TLS con PostgreSQL y muestra si el rol puede omitir RLS |
| `node scripts/verificar-db.js` | Diagnóstico rápido de conexión sin mostrar credenciales |
| `npm run dev` | Inicia la API con recarga automática (nodemon) |
| `npm start` | Inicia la API sin recarga automática |
| `npm run seed:admin` | Crea el **primer** administrador. Solo funciona si no existe ninguno; no restablece contraseñas ni eleva privilegios |

Al arrancar debe aparecer:

```text
Servidor ejecutándose en http://localhost:3000
Swagger UI: http://localhost:3000/api-docs
OpenAPI JSON: http://localhost:3000/openapi.json
```

## 4. Primer administrador

Con `ADMIN_*` configuradas en `.env`:

```powershell
npm run seed:admin
```

Después de comprobar que el login funciona, comenta o elimina las líneas `ADMIN_*` del `.env`.

## 5. Estructura

```text
hospital_backend/
├── certs/            Certificado CA de Supabase
├── database/         Scripts SQL y su README
├── scripts/          Diagnóstico de conexión
└── src/
    ├── app.js        Express: Helmet, CORS, rate limit, rutas, Swagger
    ├── config/       Pool de PostgreSQL con TLS
    ├── routes/       Endpoints y su documentación OpenAPI
    ├── middlewares/  JWT, roles, propiedad (BOLA), validadores, errores
    ├── controllers/  Lógica HTTP
    ├── services/     Consultas SQL parametrizadas
    ├── scripts/      crearAdmin.js (bootstrap)
    ├── docs/         Configuración de Swagger
    └── utils/        JWT, bcrypt, criptografía
```

## 6. Endpoints principales

| Grupo | Ruta base | Credencial |
|---|---|---|
| Login | `POST /api/auth/login` | Ninguna (rate limit: 10 intentos / 15 min) |
| Perfil, cambio de contraseña, códigos de recuperación | `/api/auth/...` | JWT |
| Recuperación administrativa | `POST /api/auth/recuperacion/admin/{usuarioId}` | JWT de administrador |
| Restablecer con token o código | `POST /api/auth/restablecer-password`, `/restablecer-con-codigo` | Ninguna (rate limit) |
| Recursos | `/api/pacientes`, `/medicos`, `/especialidades`, `/consultorios`, `/citas`, `/usuarios` | JWT + rol |
| Indicadores | `GET /api/estadisticas/resumen` | JWT de administrador |
| Cliente privado | `GET /api/seguridad/cliente` | API Key (`X-API-Key`) |

Detalle completo en Swagger: `http://localhost:3000/api-docs`.

## 7. Observaciones conocidas

- **RLS:** las 9 tablas tienen RLS habilitado, pero la conexión del backend usa el rol `postgres`, que tiene `BYPASSRLS`. La protección efectiva de los datos la dan JWT, RBAC y BOLA en Express. Pendiente para producción: usar un rol sin `BYPASSRLS`.
- **Swagger y `/api/seguridad/cliente`:** la operación no declara `ApiKeyAuth` en su documentación, así que Swagger UI no envía la cabecera `X-API-Key`. Para probarla, usar `curl.exe -H "X-API-Key: ..."` o Postman.
- **Contraseñas con caracteres no ASCII** (por ejemplo `—`): PowerShell 5 los codifica mal con `Invoke-RestMethod`. Enviar el cuerpo como bytes UTF-8 o evitar esos caracteres.

## 8. Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| `SELF_SIGNED_CERT_IN_CHAIN` | Falta el certificado CA o `sslmode` en la URL | Revisar `certs/` y quitar `?sslmode=...` de `DATABASE_URL`. No usar `rejectUnauthorized: false` |
| `ENOTFOUND` o timeout | Cadena *Direct* (IPv6) | Usar la cadena de **Session pooler** |
| 500 "Origen CORS no autorizado" | Origen no incluido | Añadirlo a `ALLOWED_ORIGIN` y reiniciar |
| 401 "Sesión revocada" | Cambió la contraseña del usuario | Iniciar sesión de nuevo |
| 429 | Rate limit | Esperar 15 minutos o reiniciar el servidor en desarrollo |
