# Microservicio: Ventas

Descripción
------------
Responsable de la gestión del ciclo comercial: creación y registro de ventas, cotizaciones, notas de crédito/débito, facturación (integración con pasarelas o ERP),(history de transacciones y reportes comerciales). Publica y consume eventos para orquestar flujos con Pedidos, Productos, Almacén, Pagos y Notificaciones.

Objetivos
---------
- Registrar ventas y cotizaciones, gestionar estados y pagos.
- Integrar con Pagos y Facturación (o exponer eventos para orquestación).
- Mantener historial, devoluciones y notas crediticias.
- Emitir eventos para sincronización (order/commercial events).
- Tests, Dockerización y despliegue independiente.
- Observabilidad: healthcheck, métricas y logs.

Stack recomendado
-----------------
- Node.js 20+, ECMAScript modules
- Express
- Sequelize (Postgres)
- Message broker (RabbitMQ / Kafka) para eventos
- Redis para cachés/idempotency (opcional)
- dotenv, Joi/express-validator
- Jest / Supertest
- Docker

Estructura de carpetas (base)
-----------------------------
/microservicios/ventas/
- src/
  - config/ (db.js, env.js, broker.js)
  - models/ (Venta.js, VentaItem.js, Cotizacion.js, Pago.js, NotaCredito.js)
  - controllers/
  - routes/
  - services/ (ventasService, facturacionService, integracionPagos)
  - middleware/ (auth, errorHandler, idempotency, rateLimit)
  - utils/ (events.js, validators.js, logger.js)
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
PORT=3003
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ventas_db
DB_USER=postgres
DB_PASS=12345
JWT_SECRET=mi_clave_secreta
BROKER_URL=amqp://localhost
REDIS_URL=redis://localhost:6379
NODE_ENV=development
IDEMPOTENCY_TTL=3600

Modelos principales
-------------------
Venta
- id, clienteId, tipo (factura/boleta/cotizacion), estado, total, impuestos, moneda, referenciaPago, createdAt, updatedAt

VentaItem
- id, ventaId, productoId, descripcion, cantidad, precioUnitario, descuento, subtotal

Cotizacion
- id, clienteId, items (json), validezHasta, estado, total

Pago
- id, ventaId, provider, providerPaymentId, amount, status, method, responseRaw, createdAt

NotaCredito / NotaDebito
- id, ventaIdOrigen, motivo, monto, itemsAfectados, estado, createdAt

Contratos / Endpoints (mínimos)
-------------------------------
- POST /ventas
  - body: { clienteId, items, tipo, direccion, paymentMethod?, metadata?, clientRequestId? }
  - response: 201 { ventaId, estado }
  - comportamiento: idempotencia por clientRequestId
- POST /ventas/cotizaciones
  - crear cotización que puede convertirse a venta
- GET /ventas/:id
- GET /ventas?clienteId=&estado=&page=
- POST /ventas/:id/cancelar
- POST /ventas/:id/generar-nota-credito
- POST /ventas/:id/confirmar-pago (webhook)
- GET /ventas/health

Flujos e integraciones
----------------------
- Reserva/consumo de stock: coordinar con microservicio Almacén (eventos o API).
- Pagos: iniciar cobros y recibir webhooks; reconciliación y estados.
- Facturación: sincronizar con ERP o servicio de facturación electrónica.
- Notificaciones: publicar eventos para envío de comprobantes y alertas.
- Emitir eventos: sale.created, sale.paid, sale.cancelled, sale.refunded

Autenticación y autorización
-----------------------------
- JWT en header Authorization: Bearer <token>
- Middleware verificarToken y checkRole(allowedRoles)
- Clientes solo acceden a sus ventas/cotizaciones; roles comerciales/administrativos pueden gestionar todas.

Lógica de negocio (services)
---------------------------
- Crear venta: validar precios actuales (Product service), crear registro, reservar stock (Almacén), procesar pago o marcar pendiente.
- Cotizaciones: generar, actualizar, convertir a venta con control de validez.
- Pagos & conciliación: procesar callbacks, actualizar estados, emitir eventos.
- Devoluciones y notas de crédito: validar reglas fiscales y actualizar stock/contabilidad.
- Idempotencia y transacciones: asegurar consistencia en operaciones distribuidas.

Validaciones y seguridad
------------------------
- Validar esquema de entrada (cantidad > 0, precios, impuestos).
- Evitar condiciones de carrera: locks o versionado optimista.
- Sanitizar metadata y campos libres.
- Rate limiting en endpoints críticos.
- Registro de auditoría para cambios sensibles.

Testing
-------
- Unit tests para servicios críticos (creación, pagos, notas).
- Integration tests (Supertest) para endpoints y flows con broker simulado.
- Tests de idempotencia y concurrencia.
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

Recomendación: docker-compose para Postgres + Broker + Redis en desarrollo.

DB Migrations / Seeders
-----------------------
- Migraciones para tablas Venta, VentaItem, Cotizacion, Pago, NotaCredito
- Seeders: estados iniciales y datos de ejemplo (clientes, productos referenciales)

Eventos y contratos de mensajería
--------------------------------
- Emitir: sale.created, sale.stock_reserved, sale.paid, sale.cancelled, sale.refunded
- Consumir: payment.confirmed, payment.failed, almacen.reservation_result, product.price_updated
- Diseñar contratos idempotentes y manejos de retry/DLQ.

Observabilidad
--------------
- Healthcheck: GET /health -> { status: "ok", db: "connected", broker: "connected" }
- Logs estructurados con correlationId/requestId
- Métricas Prometheus: ventas por estado, tiempo medio de cobro, fallos de conciliación
- Tracing recomendado (OpenTelemetry) para flows cross-service

CI / CD
-------
- Pipeline: lint -> tests -> build image -> push -> deploy
- Ejecutar migraciones y sanity checks en despliegues
- Contract tests para eventos con consumidores principales

Contrato con API Gateway
------------------------
- Prefijo: /ventas
- Gateway valida JWT y puede inyectar headers (cliente/tenant)
- Endpoints públicos sensibles no deben cachearse; descargas de comprobantes pueden usar URLs firmadas

Consideraciones adicionales
---------------------------
- Cumplir requerimientos fiscales locales para facturación electrónica (si aplica).
- Mantener trazabilidad completa para auditoría y contabilidad.
- Versionado de API: /v1/ventas
- Respuestas de error consistentes: { msg, code, details? }

Checklist de tareas a implementar
-------------------------------
- [ ] Configuración y conexión a la DB + broker (migraciones)
- [ ] Modelos: Venta, VentaItem, Cotizacion, Pago, NotaCredito
- [ ] Services: creación de venta/cotización, pagos, facturación, devoluciones
- [ ] Controllers y rutas con validaciones e idempotencia
- [ ] Middleware auth, roles e idempotency
- [ ] Tests unitarios e integrados
- [ ] Dockerfile y docker-compose de desarrollo
- [ ] Seeders (datos de prueba)
- [ ] Logging, healthcheck, métricas y tracing
- [ ] Emisión y consumo de eventos, contract tests

Referencias
-----------
Seguir convenciones del monorepo para mantener homogeneidad entre microservicios. Documentar cualquier desviación en este README.