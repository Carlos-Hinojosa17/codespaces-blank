# Microservicio: Reportes

Descripción
------------
Microservicio responsable de generación, almacenamiento y entrega de reportes y dashboards: informes operativos, resúmenes financieros, métricas y reportes ad-hoc. Consume datos de otros microservicios (ventas, pedidos, productos, almacen, usuarios) y expone APIs para solicitar, obtener y programar reportes. Soporta generación síncrona y asíncrona (jobs), exportación (CSV, XLSX, PDF) y caché.

Objetivos
---------
- Generación de reportes programados y bajo demanda.
- Exportación a CSV, XLSX y PDF.
- Persistencia y versionado de reportes generados.
- Dashboards básicos y endpoints para métricas agregadas.
- Jobs asíncronos con colas y retries.
- Seguridad, auditoría y observabilidad.

Stack recomendado
-----------------
- Node.js 20+, ECMAScript modules
- Express
- Sequelize (Postgres) o alternativa analítica
- Redis + BullMQ para colas de generación
- Puppeteer / wkhtmltopdf para PDF; exceljs / csv-stringify para export
- dotenv, Joi/express-validator
- Jest / Supertest para pruebas
- Docker

Estructura de carpetas (base)
-----------------------------
/microservicios/reportes/
- src/
  - config/ (db.js, env.js, queue.js)
  - models/ (Reporte.js, ReportJob.js, ReportTemplate.js, ReportExport.js)
  - controllers/
  - routes/
  - services/ (reportesService, exportService, scheduleService)
  - workers/ (reportWorker.js, exportWorker.js)
  - middleware/ (auth, errorHandler, rateLimit)
  - utils/ (renderers.js, aggregators.js, logger.js, validators.js)
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
PORT=3010
DB_HOST=localhost
DB_PORT=5432
DB_NAME=reportes_db
DB_USER=postgres
DB_PASS=12345
JWT_SECRET=mi_clave_secreta
REDIS_URL=redis://localhost:6379
NODE_ENV=development
PDF_RENDERER=puppeteer
EXPORT_TEMP_PATH=/tmp/exports

Modelos principales
-------------------
ReporteTemplate
- id, nombre, clave, descripción, querySpec (json), parametersSchema (json), formatoSalida (csv,xlsx,pdf,json), activo, version, createdAt, updatedAt

ReportJob
- id, templateId, parameters (json), requestedBy, estado (queued, running, completed, failed), startedAt, finishedAt, attempts

Reporte (ReporteExport)
- id, jobId, templateId, filename, url, formato, size, status, metadata, createdAt

Contratos / Endpoints (mínimos)
-------------------------------
- POST /reportes/request
  - body: { templateKey, parameters, outputFormat?, schedule? }
  - response: 202 { jobId, estado: "queued" } (soporta sync if ?sync=true)
- GET /reportes/jobs/:id
  - estado y enlaces a exportaciones
- GET /reportes/exports/:id
  - descarga del archivo generado (protegido)
- GET /reportes/templates
- POST /reportes/templates (admin)
- PUT /reportes/templates/:id (admin)
- GET /reportes/metrics?from=&to=&granularity=
  - métricas agregadas (p. ej. ventas por día/mes)
- POST /reportes/schedule
  - programar reportes periódicos (cron)
- GET /reportes/health

Autenticación y autorización
-----------------------------
- JWT en header Authorization: Bearer <token>
- Middleware verificarToken y checkRole(allowedRoles)
- Endpoints de gestión y programación restringidos a roles admin/analista
- Asegurar control de acceso a datos sensibles (scopes/tenants)

Lógica de negocio (services & workers)
-------------------------------------
- Normalizar y validar parámetros de entrada contra el template.
- Ejecutar consultas/aggregaciones contra la DB o llamar a APIs de origen.
- Generar archivos (CSV/XLSX/PDF) en workers con retries y TTL.
- Guardar metadatos del reporte y URL segura (pre-signed o proxied).
- Soportar reportes paginados, streaming para grandes volúmenes.
- Programación: usar cron jobs almacenados y cola para ejecución.
- Cache de resultados para reportes costosos con invalidación por evento.

Validaciones y seguridad
------------------------
- Validar parámetros y límites (rango de fechas, tamaño máximo).
- Evitar inyección en queries; parametrizar y/o usar query builders.
- Controlar acceso a datos por tenant/usuario.
- Encriptar o restringir URLs de descarga; expiración y logs de acceso.
- Rate limiting para endpoints públicos de generación.

Exportación y almacenamiento
----------------------------
- Archivos temporales en storage local o S3 según provider.
- TTL para archivos generados; limpieza automática por worker.
- Metadatos en DB para auditoría y re-descarga.

Testing
-------
- Unit tests para agregadores y renderers.
- Integration tests para endpoints y flujo de job (simular Redis).
- Tests de performance para export pipeline (streaming).
- Cobertura recomendada: >=80% en lógica crítica.

Docker y despliegue
-------------------
Dockerfile base:
- FROM node:20
- WORKDIR /app
- COPY package*.json ./
- RUN npm install
- COPY . .
- EXPOSE $PORT
- CMD ["node", "src/server.js"]

Recomendación: docker-compose para Postgres + Redis en desarrollo. Gestionar almacenamiento (S3) con secretos.

DB Migrations / Seeders
-----------------------
- Migraciones para tablas ReporteTemplate, ReportJob, ReporteExport.
- Seeders: templates estándar (ventas_diarias, stock_resumen, top_productos).

Integración y eventos
---------------------
- Escuchar eventos relevantes para invalidar cache o disparar reportes (ej. order.completed -> actualizar métricas).
- Emitir eventos: report.job.created, report.job.completed, report.export.available
- Diseñar contratos de eventos y retries idempotentes.

Observabilidad
--------------
- Healthcheck: GET /health -> { status: "ok", db: "connected", redis: "connected" }
- Logs estructurados (pino/winston) con correlationId/requestId
- Métricas Prometheus: jobs queued/running/completed, export sizes, latencias de generación
- Tracing (OpenTelemetry) recomendado para flows asíncronos

CI / CD
-------
- Pipeline: lint -> tests -> build image -> push -> deploy
- Ejecutar migraciones y sanity checks antes del rollout
- Monitorizar fallos en cola y alertas por backlog o DLQ

Contrato con API Gateway
------------------------
- Prefijo: /reportes
- Gateway debe validar JWT y pasar headers de tenant/usuario
- Endpoints de descarga deben ser proxied o usar URLs firmadas

Consideraciones adicionales
---------------------------
- Soporte multi-tenant si aplica (scoping en templates y datos).
- Implementar cuotas por usuario/organización para generación de reportes costosos.
- Versionado de templates y posibilidad de rollback.
- Política de retención de archivos y datos generados.
- Respuestas consistentes de error { msg, code, details? }

Comandos de desarrollo útiles
-----------------------------
- npm install
- npm run dev
- npm test
- docker-compose up -d (Postgres + Redis)
- docker build -t reportes:dev .
- docker run --env-file .env -p 3010:3010 reportes:dev

Checklist de tareas a implementar
-------------------------------
- [ ] Configuración y conexión a la DB + Redis (migraciones)
- [ ] Modelos: ReporteTemplate, ReportJob, ReportExport
- [ ] Services: generación, export, scheduling, cache
- [ ] Workers y colas con retries y DLQ
- [ ] Controllers y routes con validaciones y auth
- [ ] Tests unitarios e integrados
- [ ] Dockerfile y docker-compose de desarrollo
- [ ] Seeders (templates base)
- [ ] Logging, healthcheck, métricas y tracing
- [ ] Integración de storage (local/S3) y limpieza automática

Referencias
-----------
Seguir convenciones del monorepo para mantener homogeneidad entre microservicios. Documentar cualquier desviación en este README.