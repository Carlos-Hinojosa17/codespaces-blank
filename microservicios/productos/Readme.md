# Microservicio: Productos

Descripción
------------
Microservicio encargado del catálogo de productos: creación, edición, búsqueda, categorización, atributos, variantes, imágenes y sincronización con otros servicios (almacén, pedidos, ventas). Expone API REST para frontend y APIs internas; publica/consume eventos para mantener consistencia entre microservicios.

Objetivos
---------
- CRUD completo de productos y sus relaciones (categorías, marcas, modelos, variantes).
- Búsqueda y filtrado rápido (por texto, categoría, atributos, precio).
- Gestión de imágenes/assets y metadata (SKU, atributos, variantes).
- Soportar import/export masivo (CSV/Excel).
- Publicar eventos (product.created/updated/deleted) para sincronización con almacén, search index y caches.
- Tests, Dockerización y despliegue independiente.

Stack recomendado
-----------------
- Node.js 20+, ECMAScript modules
- Express
- Sequelize (Postgres) o alternativa (MongoDB si catálogo flexible)
- Redis para caching
- ElasticSearch / OpenSearch para búsqueda (opcional)
- multer / cloud storage para imágenes
- dotenv, Joi / express-validator
- Jest / Supertest para pruebas
- Docker

Estructura de carpetas (base)
-----------------------------
/microservicios/productos/
- src/
  - config/ (db.js, env.js, storage.js, search.js)
  - models/ (Producto.js, Categoria.js, Marca.js, Variante.js, Atributo.js, Imagen.js)
  - controllers/
  - routes/
  - services/ (productosService, searchService, importService, mediaService)
  - middleware/ (auth, errorHandler, rateLimit)
  - utils/ (validators.js, events.js, logger.js)
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
PORT=3002
DB_HOST=localhost
DB_PORT=5432
DB_NAME=productos_db
DB_USER=postgres
DB_PASS=12345
JWT_SECRET=mi_clave_secreta
REDIS_URL=redis://localhost:6379
STORAGE_PROVIDER=local|s3
S3_BUCKET=
SEARCH_URL=http://localhost:9200
NODE_ENV=development

Modelos principales
-------------------
Producto
- id, sku, nombre, descripción, precioBase, activo, visible, marcaId, modeloId, categoríaId, atributos (json), createdAt, updatedAt

Categoria
- id, nombre, slug, parentId (árbol), descripción, activo

Marca
- id, nombre, slug, activo

Variante
- id, productoId, sku, atributos (ej: talla/color), precio, stockControl (boolean), activo

Imagen
- id, productoId, varianteId?, url, storageKey, orden, altText, activo

Atributo
- id, nombre, tipo (texto, número, booleano, lista), valoresPermitidos (json)

Contratos / Endpoints (mínimos)
-------------------------------
- POST /productos
  - body: { sku?, nombre, descripcion, precioBase, categoriaId, marcaId, atributos?, variantes?, images? }
  - response: 201 producto creado
- GET /productos/:id
- PUT /productos/:id
- DELETE /productos/:id (soft delete recomendado)
- GET /productos?search=&categoria=&marca=&minPrice=&maxPrice=&page=
- POST /productos/import (CSV/Excel) — procesamiento asíncrono
- POST /productos/:id/images (upload)
- GET /categorias, POST /categorias, PUT /categorias/:id
- GET /productos/health

Autenticación y autorización
-----------------------------
- JWT en header Authorization: Bearer <token>
- Middleware verificarToken y checkRole(allowedRoles)
- Endpoints de escritura restringidos a roles admin/editor
- Lectura pública según configuración (visible flag) y permisos

Lógica de negocio (services)
---------------------------
- Crear/editar: validar unicidad de SKU, normalizar atributos y generar slugs.
- Indexar en motor de búsqueda tras cambios (async/event-driven).
- Cachear respuestas frecuentes (Redis) y invalidar on update/delete.
- Importación masiva: encolar jobs, validación por lote, reportes de errores.
- Gestión de imágenes: subir a storage, generar thumbnails, limpiar assets orphan.
- Emitir eventos: product.created, product.updated, product.deleted, product.bulk_imported

Búsqueda y filtros
------------------
- Full-text search en nombre/descripcion, boosts por campo.
- Faceting por categoría, marca y atributos.
- Paginación, ordenamiento por relevancia/precio/reciente.
- Sincronización eventual con índice de búsqueda (webhooks/events).

Validaciones y seguridad
------------------------
- Validar esquema de entrada (Joi/express-validator).
- Sanitizar campos ricos (HTML) y limitar tamaño de uploads.
- Rate limiting para endpoints públicos de búsqueda.
- Control de acceso en operaciones críticas.
- Evitar exponer datos sensibles en respuestas.

Testing
-------
- Unit tests para servicios (normalización, precios, variantes).
- Integration tests (Supertest) para endpoints CRUD y búsqueda (mock de search).
- Tests para importador masivo y manejo de errores.
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

Recomendación: docker-compose para DB, Redis y ElasticSearch en desarrollo.

DB Migrations / Seeders
-----------------------
- Migraciones para tablas Producto, Categoria, Marca, Variante, Imagen, Atributo.
- Seeders: categorías y marcas iniciales, productos de ejemplo.

Integración y eventos
---------------------
- Emitir: product.created, product.updated, product.deleted, product.import.completed
- Consumir: product.sync_request, product.price_update (si cambios desde otro servicio)
- Diseñar eventos idempotentes y contract tests para consumidores

Observabilidad
--------------
- Healthcheck: GET /health -> { status: "ok", db: "connected", search: "connected?" }
- Logs estructurados (pino/winston) con correlationId
- Métricas Prometheus: requests, indexation latency, import job stats
- Tracing opcional (OpenTelemetry) para flows cross-service

CI / CD
-------
- Pipeline: lint -> tests -> build image -> push -> deploy
- Ejecutar migraciones en despliegue controlado; validar indexación de búsqueda
- Tests de contract entre servicios que consumen eventos

Contrato con API Gateway
------------------------
- Prefijo: /productos
- Rutas públicas de búsqueda pueden ser cacheadas por Gateway/CDN
- Gateway puede validar JWT y pasar tenant/client headers si aplica

Consideraciones adicionales
---------------------------
- Separar inventario físico en microservicio Almacén; aquí mantener solo metadata de control (stockControl flag, stock estimado si es útil).
- Versionado de API: /v1/productos
- Implementar soft delete y auditoría (createdBy/updatedBy)
- Respuestas consistentes de error { msg, code, details? }

Comandos de desarrollo útiles
-----------------------------
- npm install
- npm run dev
- npm test
- docker-compose up -d (Postgres + Redis + ElasticSearch)
- docker build -t productos:dev .
- docker run --env-file .env -p 3002:3002 productos:dev

Checklist de tareas a implementar
-------------------------------
- [ ] Configuración y conexión a la DB (migraciones)
- [ ] Modelos: Producto, Categoria, Marca, Variante, Imagen, Atributo
- [ ] Services: CRUD, search indexing, importador, media handling
- [ ] Controllers y rutas con validaciones
- [ ] Middleware auth y roles
- [ ] Tests unitarios e integrados
- [ ] Dockerfile y docker-compose de desarrollo
- [ ] Seeders (categorías, marcas, productos de ejemplo)
- [ ] Logging, healthcheck, métricas y tracing
- [ ] Emisión y consumo de eventos, contract tests

Referencias
-----------
Seguir convenciones del monorepo para mantener homogeneidad entre microservicios. Documentar desviaciones en este README.