# API Hospital Segura

Aplicación web hospitalaria cliente-servidor desacoplada: **React + Vite** en el frontend, **Node.js + Express** en el backend y **PostgreSQL (Supabase)** como base de datos. Incluye autenticación JWT, autorización por roles (RBAC), protección BOLA/IDOR, recuperación de contraseñas y documentación Swagger/OpenAPI.

## Arquitectura

```text
Navegador
   │
   ▼
React 19 + Vite 8 (http://localhost:5173)
   │  fetch HTTP/JSON + Authorization: Bearer <JWT>
   ▼
Express 5 (http://localhost:3000)
   │  rutas → validadores → JWT → RBAC → BOLA/IDOR → controladores → servicios
   ▼
pg.Pool + TLS (certificado CA, rejectUnauthorized: true)
   ▼
PostgreSQL en Supabase (9 tablas, esquema public)
```

## Estructura del repositorio

```text
API_HOSPITAL/
├── README.md                    ← este archivo (visión general e instalación)
├── .gitignore
├── hospital_backend/            ← API REST Node.js + Express
│   ├── README.md                ← instalación y uso del backend
│   ├── database/
│   │   └── README.md            ← instalación y verificación de la base de datos
│   ├── certs/                   ← certificado CA público de Supabase
│   ├── scripts/                 ← diagnóstico de conexión
│   └── src/                     ← código de la API
└── API_HOSPITAL_FRONTEND/       ← SPA React + Vite + Bootstrap
    └── README.md                ← instalación y uso del frontend
```

## Requisitos

| Herramienta | Versión |
|---|---|
| Node.js | 20.19 o superior (Vite 8) |
| npm | La incluida con Node |
| Cuenta Supabase | Proyecto con PostgreSQL |
| Git | Cualquier versión reciente |

## Instalación rápida

El orden importa: **base de datos → backend → frontend**.

### 1. Base de datos

Sigue `hospital_backend/database/README.md`. En una base nueva, ejecuta en el SQL Editor de Supabase los scripts en este orden:

```text
01_esquema_hospital.sql → 02_seed_roles.sql → 03 (opcional) → 04 (revisado) → 05_verificacion_lectura.sql
```

### 2. Backend (terminal 1)

```powershell
cd hospital_backend
copy .env.example .env      # luego edita .env con tus valores
npm ci
node scripts/probar-conexion.js
npm run seed:admin          # solo la primera vez: crea el primer administrador
npm run dev
```

API en `http://localhost:3000` · Swagger en `http://localhost:3000/api-docs`.

### 3. Frontend (terminal 2)

```powershell
cd API_HOSPITAL_FRONTEND
copy .env.example .env      # VITE_API_URL=http://localhost:3000
npm ci
npm run dev
```

Aplicación en `http://localhost:5173`.

## Roles del sistema

| Rol | Puede |
|---|---|
| `administrador` | Gestionar usuarios, médicos, especialidades, pacientes, consultorios y citas; emitir tokens de recuperación administrativa; ver indicadores globales |
| `medico` | Consultar su agenda y las citas que tiene asignadas; marcarlas como atendidas |
| `paciente` | Consultar sus propias citas, solicitar y cancelar citas, ver su perfil |

El registro público siempre crea cuentas con rol `paciente`. Las cuentas de médico y administrador las crea un administrador.

## Seguridad implementada

| Control | Dónde |
|---|---|
| Contraseñas con bcrypt (12 rondas) | `hospital_backend/src/utils/password.util.js` |
| JWT con expiración y `version_sesion` (revoca tokens al cambiar la contraseña) | `src/utils/jwt.util.js`, `src/middlewares/auth.middleware.js` |
| RBAC | `src/middlewares/roles.middleware.js` |
| BOLA/IDOR (propiedad de citas, pacientes y médicos) | `src/middlewares/propiedad.middleware.js` |
| Validación de entradas | `src/middlewares/*.validator.js` (express-validator) |
| Consultas SQL parametrizadas | `src/services/*.service.js` |
| Helmet, CORS con lista de orígenes y rate limiting | `src/app.js` |
| TLS verificado hacia PostgreSQL | `src/config/database.js` |
| Tokens y códigos de recuperación guardados como hash, de un solo uso | `src/services/passwordReset.service.js`, `codigosRecuperacion.service.js` |
| API Key para integraciones privadas (no se usa en el navegador) | `src/middlewares/apiKey.middleware.js` |

## Reglas para colaborar

- **Nunca** subir archivos `.env`. Solo `.env.example` con valores ficticios.
- No colocar secretos en el frontend: todo lo que empieza por `VITE_` es visible en el navegador.
- Verificar con `git status` antes de cada commit que no aparece ningún `.env`.
- Si un secreto se publica por error, cambiarlo de inmediato (contraseña de Supabase, `JWT_SECRET`, API Keys).

## Pruebas de seguridad

El proyecto se evaluó con cuatro técnicas: **SCA** (`npm audit`), **SAST** (Semgrep), **DAST** (OWASP ZAP) y **pruebas de API** (Swagger UI). Los resultados están documentados en el informe PDF de la entrega.
