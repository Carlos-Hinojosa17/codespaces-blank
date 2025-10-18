# Microservicio: Almacén

Descripción
------------
Microservicio encargado del control de inventarios, movimientos de stock, ubicaciones y operaciones relacionadas con los almacenes físicos/virtuales. Consume y expone eventos/HTTP para integrarse con otros microservicios (productos, ventas, pedidos).

Objetivos
---------
- Control preciso de stock por almacén y ubicación.
- Registrar movimientos: entradas, salidas, ajustes, transferencias.
- Soportar reservas de stock para ventas/pedidos.
- Integración con sistema de productos y ventas (eventos y APIs).
- Pruebas, Dockerización y despliegue independiente.
- Observabilidad: healthcheck, logs, métricas.

Stack recomendado
-----------------
- Node.js 20+, ECMAScript modules
- Express
- Sequelize (Postgres) o alternativa
- RabbitMQ / Kafka para eventos (opcional)
- dotenv para configuración
- Joi / express-validator para validaciones
- Jest / Supertest para pruebas
- Docker

Estructura de carpetas (base)
-----------------------------
/microservicios/almacen/
- src/
  - config/ (db.js, env.js, messageBus.js)
  - models/ (Almacen.js, Ubicacion.js, Stock.js, Movimiento.js, Reserva.js)
  - controllers/
  - routes/
  - services/
  - middleware/ (auth, errorHandler, rateLimit)
  - utils/ (events.js, logger.js, validators.js)
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
PORT=3005
DB_HOST=localhost
DB_PORT=5432
DB_NAME=almacen_db
DB_USER=postgres
DB_PASS=12345
JWT_SECRET=mi_clave_secreta
NODE_ENV=development
MESSAGE_BROKER_URL=amqp://localhost

Modelos principales
-------------------
Almacen
- id, nombre, dirección, codigo, activo, createdAt, updatedAt

Ubicacion
- id, almacénId, código (ej: A1-01), descripción, tipo, capacidad

Stock
- id, productoId, almacenId, ubicacionId (opcional), cantidad, lote/serie (opcional), estado

Movimiento
- id, tipo (entrada/salida/ajuste/transferencia), productoId, cantidad, origen (alm/ubic), destino (alm/ubic), referencia, usuarioId, fecha

Reserva
- id, pedidoId/ventaId, productoId, cantidad, estado (activa/consumida/cancelada), expiración

Contratos / Endpoints (mínimos)
-------------------------------
- POST /almacen/stock/check
  - body: [{ productoId, cantidad, almacenId }]
  - response: disponibilidad por item
- POST /almacen/stock/reservar
  - body: { pedidoId, items: [{ productoId, cantidad, almacenId }] }
  - response: reservas creadas | error si no hay stock
- POST /almacen/movimientos
  - body: { tipo, productoId, cantidad, origen, destino, referencia }
  - response: movimiento creado
- GET /almacen/stock/:productoId
  - query: almacenId, ubicacionId
- GET /almacen/almacenes
- POST /almacen/transferencia
  - cuerpo: origenAlmacen, destinoAlmacen, items
- GET /almacen/reservas/:pedidoId
- GET /almacen/health

Autenticación y autorización
-----------------------------
- JWT en header Authorization: Bearer <token>
- Middleware verificarToken y checkRole(allowedRoles)
- Rutas sensibles (ajustes, transferencias masivas) solo admin/operaciones

Lógica de negocio (services)
---------------------------
- Comprobación de stock disponible considerando reservas y pedidos confirmados.
- Reservas: crear, confirmar (consumir), cancelar con expiración automática.
- Movimientos: registrar y actualizar stock con transacción DB.
- Transferencias entre almacenes con consistencia (transacción).
- Ajustes y conciliaciones periódicas.
- Emisión y consumo de eventos (ej. producto creado/actualizado, pedido confirmado).

Validaciones y seguridad
------------------------
- Validar entradas (cantidad positiva, existencia de producto/almacén).
- Operaciones de stock en transacciones DB para evitar race conditions.
- Rate limiting en endpoints públicos.
- Sanitización de datos y control de permisos.
- Logs de auditoría para movimientos (usuario, timestamp, referencia).

Testing
-------
- Unit tests para services críticos (reservas, movimientos, conciliación).
- Integration tests (Supertest) para endpoints de reserva y movimiento.
- Simular concurrencia en pruebas de stock.
- Cobertura mínima recomendada: 80% en lógica de negocio crítica.

Docker y despliegue
-------------------
Dockerfile base y recomendación de docker-compose para DB y broker:
- FROM node:20
- COPY package*.json, npm install, COPY src, EXPOSE $PORT, CMD ["node","src/server.js"]

Recomendación: correr migraciones y seeders en pasos controlados en CI/CD.

DB Migrations / Seeders
-----------------------
- Migraciones para tablas Almacén, Ubicación, Stock, Movimiento, Reserva.
- Seeders: almacén inicial, ubicaciones básicas.

Integración y eventos
---------------------
- Emitir eventos: stock.reservado, stock.liberado, movimiento.creado, transferencia.realizada.
- Consumir eventos relevantes: pedido.confirmado, pedido.cancelado, producto.eliminado.
- Diseñar contratos de eventos y retries idempotentes.

Observabilidad
--------------
- Healthcheck endpoint: GET /health -> { status: "ok", db: "connected", broker: "connected" }
- Logging estructurado (pino/winston)
- Métricas Prometheus: movimientos por tipo, stock total, reservas activas
- Tracing opcional (OpenTelemetry)

CI / CD
-------
- Pipeline: lint -> tests -> build image -> push -> deploy
- Ejecutar migraciones y checks de integridad antes de promover
- Monitoreo post-deploy para detectar desincronización de stock

Contrato con API Gateway
------------------------
- Prefijo de ruta: /almacen
- Gateway puede validar JWT; microservicio debe validar permisos críticos
- Documentar rutas y eventos para frontend y otros servicios

Consideraciones adicionales
---------------------------
- Implementar soft delete y auditoría por movimiento.
- Manejo de lotes/series y fechas de vencimiento si aplica.
- Reconciliación periódica contra inventarios físicos (ajustes).
- Versionado de API: /v1/almacen
- Respuestas consistentes de error { msg, code, details? }

Comandos de desarrollo útiles
-----------------------------
- npm install
- npm run dev (nodemon)
- npm test
- docker build -t almacen:dev .
- docker run --env-file .env -p 3005:3005 almacen:dev

Checklist de tareas a implementar
-------------------------------
- [ ] Configuración y conexión a la DB (migraciones)
- [ ] Modelos: Almacen, Ubicacion, Stock, Movimiento, Reserva
- [ ] Services: comprobación stock, reservas, movimientos, transferencias
- [ ] Controllers y rutas con validaciones
- [ ] Middleware auth y roles
- [ ] Tests unitarios e integrados (con concurrencia simulada)
- [ ] Dockerfile y docker-compose de desarrollo (DB + broker)
- [ ] Seeders (almacenes/ubicaciones)
- [ ] Logging, healthcheck y metrics
- [ ] Contratos de eventos y manejo idempotente

Referencias
-----------
Seguir las convenciones del monorepo para mantener homogeneidad entre microservicios. Documentar cualquier desviación en este README.