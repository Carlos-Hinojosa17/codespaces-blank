# Microservicio: Pedidos

Descripción
------------
Microservicio responsable de la gestión del ciclo de vida de pedidos: creación, validación, pagos, estado, cancelación, historial y coordinación con almacén, pagos y notificaciones. Expone API para frontend y otros servicios; publica/consume eventos para la orquestación.

Objetivos
---------
- Crear y validar pedidos desde la Página Web o Sistema Web.
- Orquestar reservas de stock (Almacén) y procesar pagos.
- Gestionar estados del pedido (pendiente, reservado, pagado, en_preparacion, enviado, completado, cancelado).
- Soportar idempotencia, transacciones y conciliación en caso de fallos.
- Pruebas, Dockerización y despliegue independiente.
- Observabilidad y retry/recovery de flujos críticos.

Stack recomendado
-----------------
- Node.js 20+, ECMAScript modules
- Express
- Sequelize (Postgres)
- Message broker (RabbitMQ / Kafka) para eventos
- Redis para idempotencia/locks (opcional)
- dotenv, Joi/express-validator
- Jest / Supertest
- Docker

Estructura de carpetas (base)
-----------------------------
/microservicios/pedidos/
- src/
  - config/ (db.js, env.js, broker.js)
  - models/ (Pedido.js, PedidoItem.js, Pago.js, Direccion.js, PedidoEstado.js)
  - controllers/
  - routes/
  - services/ (pedidosService, pagosService, integracionAlmacen)
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
PORT=3004
DB_HOST=localhost
DB_PORT=5432
DB_NAME=pedidos_db
DB_USER=postgres
DB_PASS=12345
JWT_SECRET=mi_clave_secreta
BROKER_URL=amqp://localhost
REDIS_URL=redis://localhost:6379
NODE_ENV=development
IDEMPOTENCY_TTL=3600

Modelos principales
-------------------
Pedido
- id, clienteId, estado, total, currency, direccionId, metadata, createdAt, updatedAt

PedidoItem
- id, pedidoId, productoId, cantidad, precioUnitario, descuento, subtotal

Pago
- id, pedidoId, provider, providerPaymentId, amount, status, method, responseRaw, createdAt

Direccion
- id, pedidoId, tipo (envio/factura), calle, ciudad, postal, pais

PedidoEstado (historial)
- id, pedidoId, estado, notas, usuarioId, timestamp

Contratos / Endpoints (mínimos)
-------------------------------
- POST /pedidos
  - body: { clienteId, items: [{ productoId, cantidad }], direccion, paymentMethod?, metadata? }
  - response: 201 { pedidoId, estado }
  - comportamiento: idempotencia por clientRequestId; validar stock/reservar luego procesar pago si aplica
- GET /pedidos/:id
  - detalla pedido, items, pagos, historial de estados
- GET /pedidos?clienteId=&estado=&page=
- POST /pedidos/:id/cancel
  - cancelar pedido: validar estado y revertir reservas/pagos según reglas
- POST /pedidos/:id/confirmar-pago (webhook o callback)
  - actualizar estado tras confirmación del proveedor de pagos
- POST /pedidos/:id/confirmar-envio
- GET /pedidos/health

Integración con otros servicios
-------------------------------
- Almacén: reservar stock al crear pedido; confirmar consumo al pago o envío; liberar reservas al cancelar.
- Pagos: iniciar cobro, recibir webhooks, reconciliar pagos fallidos.
- Notificaciones: emitir eventos para enviar emails/SMS sobre estado.
- Productos: validar precio y disponibilidad al crear pedido.

Autenticación y autorización
-----------------------------
- JWT en header Authorization: Bearer <token>
- Middleware verificarToken y checkRole(allowedRoles)
- Clientes solo pueden ver/gestionar sus pedidos; admins pueden gestionar todos

Lógica de negocio (services)
---------------------------
- Flujo atómico recomendado (transacciones y eventos):
  1. Validar items y precios (producto service).
  2. Crear pedido en estado "pendiente".
  3. Reservar stock (solicitud a Almacén). Si falla, revertir.
  4. Iniciar pago (si aplica). Si pago requerido y falla, marcar "payment_failed" y liberar reservas.
  5. Al confirmar pago: marcar "pagado", consumir reserva y emitir eventos order.paid -> fulfillment.
- Idempotencia: soportar retries para creación y callbacks de pago.
- Retries y compensaciones: definir sagas locales o orquestadas via events.

Validaciones y seguridad
------------------------
- Validar esquema de items, cantidades > 0 y límites.
- Proteger endpoints con rate limiting y checks de ownership.
- Evitar cambios concurrentes sobre un mismo pedido (locks o versionado optimista).
- Sanitizar metadata y entradas.

Testing
-------
- Unit tests para services: creación, reservas, cancelaciones y reconciliación.
- Integration tests (Supertest) para endpoints principales y flows con brokers simulados.
- Tests de concurrencia para reservas/pagos.
- Cobertura recomendada: >=80% en lógica crítica.

Docker y despliegue
-------------------
Dockerfile base: node:20, COPY, npm install, COPY src, EXPOSE $PORT, CMD ["node","src/server.js"]
Recomendación: usar docker-compose en desarrollo para levantar Postgres, Broker y Redis.

DB Migrations / Seeders
-----------------------
- Migraciones para tablas Pedido, PedidoItem, Pago, Direccion, PedidoEstado.
- Seeder: estados iniciales y datos de prueba (ej. pedidos de ejemplo).

Eventos y contratos de mensajería
--------------------------------
- Emitir: order.created, order.stock_reserved, order.stock_release, order.paid, order.cancelled, order.shipped
- Consumir: payment.confirmed, payment.failed, almacen.reservation_result, product.price_updated
- Diseñar contratos de eventos claros e idempotentes; manejar retries y dead-letter.

Observabilidad
--------------
- Healthcheck: GET /health -> { status: "ok", db: "connected", broker: "connected" }
- Logs estructurados con correlationId/requestId
- Métricas Prometheus: pedidos por estado, latencia de creación, reservas fallidas
- Tracing (OpenTelemetry) recomendado para flows cross-service

CI / CD
-------
- Pipeline: lint -> tests -> build image -> push -> deploy
- Ejecutar migraciones y checks de integridad en despliegue
- Pruebas de contratos para eventos antes de promover

Contrato con API Gateway
------------------------
- Prefijo: /pedidos
- Gateway valida JWT y puede inyectar headers (cliente/tenant)
- Endpoints públicos con caching limitado (no cachear respuestas dinámicas)

Consideraciones adicionales
---------------------------
- Soporte de pedidos parciales y backorders si aplica.
- Política de cancelación y reembolso documentada.
- Versionado de API: /v1/pedidos
- Responses consistentes de error: { msg, code, details? }

Comandos de desarrollo útiles
-----------------------------
- npm install
- npm run dev
- npm test
- docker-compose up -d (Postgres + Broker + Redis)
- docker build -t pedidos:dev .
- docker run --env-file .env -p 3004:3004 pedidos:dev

Checklist de tareas a implementar
-------------------------------
- [ ] Configuración y conexión a la DB y broker (migraciones)
- [ ] Modelos: Pedido, PedidoItem, Pago, Direccion, PedidoEstado
- [ ] Services: creación de pedido, reservas, pagos, cancelaciones, reconciliación
- [ ] Controllers y rutas con validaciones y idempotencia
- [ ] Middleware auth, roles e idempotency
- [ ] Tests unitarios e integrados (con simulación de broker/almacén)
- [ ] Dockerfile y docker-compose de desarrollo
- [ ] Seeders (datos de prueba)
- [ ] Logging, healthcheck, métricas y tracing
- [ ] Emisión y consumo de eventos, contract tests

Referencias
-----------
Seguir convenciones del monorepo para mantener homogeneidad entre microservicios. Documentar cualquier desviación en este README.