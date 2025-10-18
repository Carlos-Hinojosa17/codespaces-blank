# Microservicio: Usuarios

Descripción
------------
Microservicio responsable del registro, autenticación, autorización y gestión de perfiles/roles de usuarios que consumirán el Sistema Web (interno). Expone API REST pensada para ser proxyada por el API Gateway.

Objetivos
---------
- Registro y login con JWT.
- Gestión de roles y permisos.
- Endpoints CRUD para usuarios (con validaciones).
- Integración con base de datos independiente.
- Tests automatizados, Dockerización y despliegue independiente.
- Buenas prácticas: logging, manejo de errores, rate limit básico y seguridad.

Stack recomendado
-----------------
- Node.js 20+, ECMAScript modules
- Express
- Sequelize (Postgres) o alternativa ORM
- bcryptjs para passwords
- jsonwebtoken para JWT
- dotenv para configuración
- http-proxy-middleware (a nivel de API Gateway)
- Jest / Supertest para pruebas
- Docker

Estructura de carpetas (base)
-----------------------------
/microservicios/usuarios/
- src/
  - config/ (db.js, env.js)
  - models/ (Usuario.js, Rol.js, UsuarioRol.js si se requiere many-to-many)
  - controllers/
  - routes/
  - services/
  - middleware/ (auth, errorHandler, rateLimit)
  - utils/ (jwt.js, logger.js, validators.js)
  - app.js
  - server.js
- tests/
- migrations/
- seeders/
- package.json
- .env
- Dockerfile
- README.md

Variables de entorno (.env)
--------------------------
Ejemplo mínimo:
PORT=3001
DB_HOST=localhost
DB_PORT=5432
DB_NAME=usuarios_db
DB_USER=postgres
DB_PASS=12345
JWT_SECRET=mi_clave_secreta
JWT_EXPIRES_IN=1h
BCRYPT_SALT_ROUNDS=10
NODE_ENV=development

Modelos principales
-------------------
Usuario
- id: integer PK autoincrement
- nombre: string (not null)
- email: string (unique, not null)
- password: string (hash)
- rol: string (opcional) o relación many-to-many
- estado: boolean (activo/inactivo)
- createdAt, updatedAt

Rol (opcional)
- id, nombre, permisos (json o relación)

Contratos / Endpoints (mínimos)
-------------------------------
- POST /users/register
  - body: { nombre, email, password, rol? }
  - responses: 201 usuario creado | 400 errores
- POST /users/login
  - body: { email, password }
  - responses: 200 { token } | 401
- GET /users/ (protegido, roles: admin)
  - query: paginate, search
- GET /users/:id (protegido)
- PUT /users/:id (protegido)
- DELETE /users/:id (protegido, soft delete recomendado)
- GET /users/me (protegido) — info del token

Autenticación y autorización
-----------------------------
- JWT en header Authorization: Bearer <token>
- Middleware verificarToken que valida token y agrega req.user
- Middleware checkRole(allowedRoles) para verificar permisos
- Tokens con expiración; endpoints de refresh si se requiere

Lógica de negocio (services)
---------------------------
- Registrar: validar existencia, hashear contraseña, crear usuario, enviar evento (opcional)
- Login: verificar email, comparar password, generar JWT
- Actualizar usuario: validar cambios, control de campos sensibles
- Gestión de roles: crear/editar/asignar roles

Validaciones y seguridad
------------------------
- Validar entrada (express-validator o Joi)
- Limitar intentos de login (rate limiter)
- Almacenar sólo hashes de contraseña (bcrypt)
- Sanitizar entradas para prevenir inyección
- Forzar HTTPS en producción (config en Gateway / reverse proxy)
- Configurar políticas CORS desde API Gateway

Testing
-------
- Unit tests para services y utils
- Integration tests (Supertest) para endpoints importantes: register, login, protected routes
- Cobertura mínima recomendada: 80%

Docker y despliegue
-------------------
Dockerfile base:
- FROM node:20
- COPY package*.json, npm install, COPY src, EXPOSE $PORT, CMD ["node","src/server.js"]

Recomendación: usar docker-compose para levantar DB y migraciones en desarrollo.

DB Migrations / Seeders
-----------------------
- Usar Sequelize-CLI o herramienta de migraciones
- Seeder inicial: crear rol admin y usuario administrador con contraseña segura (documentar contraseña temporal)

Observabilidad
--------------
- Logging estructurado (p. ej. pino/winston)
- Healthcheck endpoint: GET /health -> { status: "ok", db: "connected" }
- Endpoints métricos si se requiere (Prometheus)
- Integrar tracing si procede (Jaeger, OpenTelemetry)

CI / CD
-------
- Pipeline: lint -> tests -> build image -> push -> deploy
- Ejecutar migraciones en despliegue controlado
- Escanear dependencias por vulnerabilidades

Contrato con API Gateway
------------------------
- Prefijo de ruta: /usuarios
- Gateway debe validar JWT y reenviar claims (o microservicio puede validar)
- Documentar rutas y códigos de estado para el equipo de frontend

Consideraciones adicionales
---------------------------
- Implementar soft delete y auditoría (createdBy/updatedBy)
- Versionado de API: /v1/users
- Manejar errores con respuestas consistentes { msg, code, details? }
- Documentación OpenAPI/Swagger mínima para endpoints públicos

Comandos de desarrollo útiles
-----------------------------
- npm install
- npm run dev (con nodemon)
- npm test
- npm run lint
- docker build -t usuarios:dev .
- docker run --env-file .env -p 3001:3001 usuarios:dev

Checklist de tareas a implementar
-------------------------------
- [ ] Configuración y conexión a la DB (migraciones)
- [ ] Modelos Usuario y Rol
- [ ] Services: registrar, login, CRUD
- [ ] Controllers y rutas con validaciones
- [ ] Middleware auth y roles
- [ ] Tests unitarios e integrados
- [ ] Dockerfile y docker-compose de desarrollo
- [ ] Seeders (admin/roles)
- [ ] Logging, healthcheck y metrics
- [ ] Documentación OpenAPI

Contacto y referencias
----------------------
Seguir las convenciones descritas en la raíz del proyecto para mantener homogeneidad entre microservicios. Documentar cualquier desviación en este README.