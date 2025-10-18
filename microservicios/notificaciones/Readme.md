# Microservicio: Notificaciones

Descripción
------------
Microservicio encargado del envío y gestión de notificaciones (email, SMS, push, webhooks). Actúa como capa de entrega confiable: templates, colas, reintentos, auditoría y métricas. Se integra con otros microservicios (ventas, pedidos, usuarios, configuración) mediante HTTP y eventos.

Objetivos
---------
- Envío fiable de notificaciones multi-canal (email, SMS, push, webhook).
- Plantillas versionadas y personalizables.
- Reintentos con backoff y dead-letter queue.
- Registro de intentos y estado de entrega (audit trail).
- Exposición de API para encolar notificaciones y gestionar templates.
- Tests, Dockerización y despliegue independiente.

Stack recomendado
-----------------
- Node.js 20+, ECMAScript modules
- Express
- Sequelize (Postgres) o alternativa
- BullMQ / Redis para colas y reintentos
- Nodemailer (SMTP), Twilio (SMS), Firebase Admin (push), o integraciones con SendGrid/SES
- dotenv para configuración
- Joi / express-validator para validaciones
- Jest / Supertest para pruebas
- Docker

Estructura de carpetas (base)
-----------------------------
/microservicios/notificaciones/
- src/
  - config/ (db.js, env.js, queue.js, providers.js)
  - models/ (Notificacion.js, Template.js, DeliveryAttempt.js, ChannelConfig.js)
  - controllers/
  - routes/
  - services/ (enqueueService, deliveryService, templateService)
  - workers/ (emailWorker.js, smsWorker.js, pushWorker.js)
  - middleware/ (auth, errorHandler, rateLimit)
  - utils/ (renderTemplate.js, logger.js, validators.js, idempotency.js)
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
PORT=3008
DB_HOST=localhost
DB_PORT=5432
DB_NAME=notificaciones_db
DB_USER=postgres
DB_PASS=12345
REDIS_URL=redis://localhost:6379
JWT_SECRET=mi_clave_secreta
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=user
SMTP_PASS=pass
TWILIO_SID=
TWILIO_TOKEN=
FIREBASE_CREDENTIALS_PATH=/path/to/firebase.json
NODE_ENV=development
DEFAULT_FROM_EMAIL=no-reply@example.com
MAX_RETRIES=5

Modelos principales
-------------------
Notificacion
- id, tipo (email/sms/push/webhook), templateId (opcional), payload (json), destinatarios (json), estado (queued/sent/failed/canceled), metadata, createdAt, updatedAt

Template
- id, nombre, canal, subject, body (handlebars/markdown), variablesSchema (json), version, activo, createdAt, updatedAt

DeliveryAttempt
- id, notificacionId, canal, response, success (boolean), intentoNumero, timestamp, errorCode

ChannelConfig
- id, canal, provider, credentials (encrypted), active, createdAt, updatedAt

Contratos / Endpoints (mínimos)
-------------------------------
- POST /notifications/enqueue
  - body: { tipo, canal, templateId?, destinatarios, payload, metadata? }
  - response: 202 { id, estado: "queued" }
- POST /notifications/send (sin encolar — opcional para sync)
- GET /notifications/:id
  - respuesta: detalle con attempts
- GET /notifications?estado=&tipo=&page=
- POST /templates
- GET /templates/:id
- PUT /templates/:id
- DELETE /templates/:id (soft delete)
- POST /channels/config (protegido, admin)
- GET /health

Autenticación y autorización
-----------------------------
- JWT en header Authorization: Bearer <token>
- Middleware verificarToken y checkRole(allowedRoles)
- Endpoint de gestión (templates, channels) restringidos a roles administrativos

Lógica de negocio (services & workers)
-------------------------------------
- Encolar notificaciones en Redis/BullMQ para procesamiento asíncrono.
- Workers dedicados por canal con manejo de reintentos exponencial y backoff.
- Idempotencia: evitar duplicados usando requestId/messageId.
- Persistencia de estado y delivery attempts en DB.
- Dead-letter: mover notificaciones fallidas tras MAX_RETRIES a DLQ y emitir evento.
- Renderizado de templates con validación del schema de variables.
- Webhooks: firma/verificación y reintentos con jitter.
- Emitir eventos: notification.sent, notification.failed, notification.queued

Validaciones y seguridad
------------------------
- Validar esquema de payload para templates (Joi/JSON Schema).
- Encriptar credenciales de proveedores en DB.
- Rate limiting por origen y por destino (para SMS/email).
- Sanitizar datos para evitar inyección en templates.
- Proteger endpoints de gestión con roles.
- Limitar tamaño de mensajes y attachments.

Testing
-------
- Unit tests para renderizado de templates y lógica de reintentos.
- Integration tests para endpoints de encolado y procesamiento (simular Redis).
- Tests de contratos para proveedores (mock SMTP/Twilio/Firebase).
- Cobertura recomendada: >=80% en lógica crítica (workers, reintentos, idempotencia).

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

Recomendación: docker-compose para levantar Redis y Postgres en desarrollo. Gestionar credenciales sensibles con secretos del orquestador.

DB Migrations / Seeders
-----------------------
- Migraciones para tablas Notificacion, Template, DeliveryAttempt, ChannelConfig.
- Seeders: templates básicos (email de bienvenida), configuración de canal (si aplica).

Integración y eventos
---------------------
- Consumir eventos relevantes: user.registered, order.placed, payment.failed
- Emitir eventos: notification.queued, notification.sent, notification.failed
- Asegurar contratos de eventos y idempotencia al procesar eventos externos

Observabilidad
--------------
- Healthcheck: GET /health -> { status: "ok", db: "connected", redis: "connected" }
- Métricas Prometheus: notificaciones por canal, tasa de éxito, reintentos, DLQ size
- Logs estructurados (pino/winston) con correlationId/requestId
- Tracing opcional (OpenTelemetry) para seguimiento across services

Errores y dead-letter handling
------------------------------
- Registrar errorCodes y respuestas crudas del proveedor en DeliveryAttempt.
- Mecanismo de notificación/alerta cuando DLQ supera umbral.
- Endpoint/worker para reintentar manualmente items en DLQ.

CI / CD
-------
- Pipeline: lint -> tests -> build image -> push -> deploy
- Validar migraciones y sanity checks antes de promover
- Monitoreo post-deploy y alertas por errores en DLQ

Contrato con API Gateway
------------------------
- Prefijo: /notificaciones
- Gateway puede validar JWT y pasar correlation headers
- Rutas de encolado públicas (internal services) deben autenticarse con JWT de servicio

Consideraciones adicionales
---------------------------
- Soporte attachments para email con límites y almacenamiento temporal.
- Implementar throttling por proveedor para evitar bloqueos.
- Versionado de templates y rollback fácil a versiones previas.
- Respuestas consistentes de error { msg, code, details? }

Comandos de desarrollo útiles
-----------------------------
- npm install
- npm run dev
- npm test
- docker-compose up -d (Postgres + Redis)
- docker build -t notificaciones:dev .
- docker run --env-file .env -p 3008:3008 notificaciones:dev

Checklist de tareas a implementar
-------------------------------
- [ ] Configuración y conexión a la DB + Redis (migraciones)
- [ ] Modelos: Notificacion, Template, DeliveryAttempt, ChannelConfig
- [ ] Services: enqueue, delivery, template rendering, idempotency
- [ ] Workers por canal con reintentos y DLQ
- [ ] Controllers y rutas con validaciones
- [ ] Middleware auth y roles
- [ ] Tests unitarios e integrados (simular proveedores)
- [ ] Dockerfile y docker-compose (Postgres + Redis)
- [ ] Seeders (templates base)
- [ ] Logging, healthcheck, métricas y tracing
- [ ] Emisión y consumo de eventos, contract tests

Referencias
-----------
Seguir convenciones del monorepo para mantener homogeneidad entre microservicios. Documentar cualquier desviación en este README.