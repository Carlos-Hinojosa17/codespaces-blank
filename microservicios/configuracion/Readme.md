# Microservicio: Configuración

Descripción
------------
Microservicio responsable de la configuración global de la plataforma: branding (logos, colores), parámetros de negocio, feature flags, textos y ajustes de la aplicación que consumen el Sistema Web y la Página Web.

Objetivos
---------
- Gestionar parámetros globales (nombre, logo, colores, moneda, timezone).
- Gestionar feature flags y entornos por cliente/tenant.
- Exponer assets (logos, favicon) y configuraciones versionadas.
- Permitir cambios seguros y controlados (auditoría, roles).
- Proveer endpoints para frontend y para otros microservicios.

Stack recomendado
-----------------
- Node.js 20+, ECMAScript modules
- Express
- Sequelize (Postgres) o MongoDB según necesidad
- multer / cloud storage para assets
- jsonwebtoken, dotenv
- Joi / express-validator
- Jest / Supertest
- Docker

Estructura de carpetas (base)
-----------------------------
/microservicios/configuracion/
- src/
  - config/ (db.js, env.js, storage.js)
  - models/ (Configuracion.js, Asset.js, FeatureFlag.js, AuditLog.js)
  - controllers/
  - routes/
  - services/
  - middleware/ (auth, roles, errorHandler)
  - utils/ (validators.js, storageClients.js)
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
PORT=3007
DB_HOST=localhost
DB_PORT=5432
DB_NAME=config_db
DB_USER=postgres
DB_PASS=12345
JWT_SECRET=mi_clave_secreta
STORAGE_PROVIDER=local|s3
S3_BUCKET=
S3_REGION=
NODE_ENV=development

Modelos principales
-------------------
Configuracion
- id, clave (ej: site.name), valor (json/text), ambiente, scope (global/tenant), versión, activo, createdAt, updatedAt

Asset
- id, nombre, tipo (logo,favicon), url, storageKey, tamaño, mime, activo

FeatureFlag
- id, clave, enabled, rules (por tenant/rol), descripción

AuditLog
- id, recurso, acción, usuarioId, payloadAntes, payloadDespues, timestamp

Contratos / Endpoints (mínimos)
-------------------------------
- GET /configuracion/:clave
  - query: ambiente, tenant
  - respuesta: valor
- GET /configuracion (protegido)
  - query: filtro, versión
- POST /configuracion (protegido, roles admin)
  - body: { clave, valor, ambiente, scope }
- PUT /configuracion/:id (protegido)
- DELETE /configuracion/:id (soft delete)
- POST /configuracion/assets (upload de logos/assets)
- GET /configuracion/assets/:id
- GET /configuracion/feature-flags
- POST /configuracion/feature-flags (gestión de flags)
- GET /configuracion/health

Autenticación y autorización
-----------------------------
- JWT en header Authorization: Bearer <token>
- Middleware verificarToken y checkRole(allowedRoles)
- Operaciones de escritura restringidas a roles administrativos
- Registrar auditoría en cambios sensibles

Lógica de negocio (services)
---------------------------
- Versionado y auditoría de cambios en configuraciones.
- Resolución de configuración por ambiente y scope (tenant > ambiente > global).
- Validación de schemas para valores JSON.
- Upload de assets a storage local o S3 y generación de URLs públicas/firmadas.
- Evaluación de feature flags con reglas (por tenant, porcentaje, roles).

Validaciones y seguridad
------------------------
- Validar esquemas JSON para configuraciones complejas.
- Sanitizar y limitar tipos/tamaños de assets.
- Control de acceso por roles y scope.
- Almacenar sólo referencias a assets (no binarios en DB).
- Rate limiting en endpoints públicos de lectura si aplica.

Testing
-------
- Unit tests para resolución de configuración y evaluación de flags.
- Integration tests para endpoints de assets y CRUD.
- Tests de seguridad: subida de archivos, permisos.
- Cobertura recomendada: >=80% en lógica crítica.

Docker y despliegue
-------------------
- Dockerfile base: node:20, copiar, npm install, exponer $PORT, CMD ["node","src/server.js"]
- Recomendar docker-compose para DB y almacenamiento en desarrollo.
- Migraciones y seeders controlados en CI/CD.

DB Migrations / Seeders
-----------------------
- Migraciones para tablas Configuracion, Asset, FeatureFlag, AuditLog.
- Seeder inicial con configuración base (site.name, moneda, timezone).

Integración y eventos
---------------------
- Emitir eventos: configuracion.actualizada, asset.subido, featureflag.cambiado.
- Consumir eventos relevantes si otros servicios necesitan refrescar caché.
- Diseñar eventos idempotentes y contract tests si procede.

Observabilidad
--------------
- Healthcheck: GET /health -> { status: "ok", db: "connected", storage: "ok" }
- Logging estructurado (pino/winston)
- Cache headers y ETag para endpoints públicos de configuración
- Métricas: cambios por usuario, tamaño de assets subidos

CI / CD
-------
- Pipeline: lint -> tests -> build image -> push -> deploy
- Ejecutar migraciones antes del rollout
- Validaciones de contract tests si se publican eventos

Contrato con API Gateway
------------------------
- Prefijo: /configuracion
- Rutas públicas de solo lectura pueden ser cacheadas por el Gateway/CDN
- Gateway puede inyectar tenant en headers para resolución de scope

Consideraciones adicionales
---------------------------
- Soporte multi-tenant si aplica (scoping en configuraciones).
- TTL/Cache para configuraciones en servicios consumidores.
- Proveer endpoint para invalidar cache distribuida.
- Versionar API: /v1/configuracion
- Documentar cambios en configuración crítica.

Comandos de desarrollo útiles
-----------------------------
- npm install
- npm run dev
- npm test
- docker build -t configuracion:dev .
- docker run --env-file .env -p 3007:3007 configuracion:dev

Checklist de tareas a implementar
-------------------------------
- [ ] Configuración y conexión a la DB (migraciones)
- [ ] Modelos: Configuracion, Asset, FeatureFlag, AuditLog
- [ ] Services: resolución, versionado, assets, flags
- [ ] Controllers y rutas con validaciones y auditoría
- [ ] Middleware auth y roles
- [ ] Tests unitarios e integrados
- [ ] Dockerfile y docker-compose (DB + storage)
- [ ] Seeders con configuración base
- [ ] Logging, healthcheck y caching
- [ ] Emisión de eventos y contract tests

Referencias
-----------
Seguir convenciones del monorepo para mantener homogeneidad entre microservicios. Documentar desviaciones en este README.