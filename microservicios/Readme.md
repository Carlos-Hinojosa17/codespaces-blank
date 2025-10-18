# Monorepo: Microservicios

Descripción
------------
Carpeta raíz que agrupa los microservicios del sistema: cada servicio es independiente, tiene su propio README, configuración y despliegue. Este README central documenta convenciones, servicios disponibles y pasos comunes para desarrollo y despliegue.

Servicios incluidos
-------------------
- usuarios — Auth, roles, JWT (PORT 3001)  
- productos — Catálogo, variantes, búsqueda (PORT 3002)  
- ventas — Registro de ventas y cotizaciones (PORT 3003)  
- pedidos — Gestión del ciclo de pedidos (PORT 3004)  
- almacen — Control de inventario y reservas (PORT 3005)  
- reportes — Generación y exportación de reportes (PORT 3010)  
- configuracion — Branding, feature flags, parámetros (PORT 3007)  
- notificaciones — Email/SMS/push/queues (PORT 3008)

Cada microservicio tiene su propio README.md con detalles de modelos, endpoints y checklist.

Convenciones del monorepo
-------------------------
- Stack recomendado: Node.js 20+ (ESM), Express, Sequelize (Postgres).  
- Cada servicio contiene:
  - src/, tests/, migrations/, seeders/, package.json, .env, Dockerfile, README.md
  - app.js / server.js como entrada
  - config/, models/, controllers/, services/, routes/, middleware/, utils/
- Variables de entorno por servicio: mantener archivo .env local (NO subir secretos).
- API Gateway: prefijos por servicio (/usuarios, /productos, /ventas, /pedidos, /almacen, /reportes, /configuracion, /notificaciones).
- Versionado de API: usar prefijo /v1/ cuando se introduzcan breaking changes.

Buenas prácticas
----------------
- Salud y observabilidad: exponer GET /health, logging estructurado (pino/winston), métricas (Prometheus) y tracing (OpenTelemetry) cuando sea posible.
- Seguridad: JWT en Authorization Bearer, rate limiting, sanitización y validación de inputs (Joi/express-validator).
- Errores: respuesta consistente { msg, code, details? } y middleware global de manejo de errores.
- DB: usar migraciones y seeders; no ejecutar sync() en producción.
- Transacciones: usar transacciones DB para operaciones críticas y sagas/eventos para flows distribuidos.
- Idempotencia: endpoints críticos (creación de pedidos/ventas) deben aceptar clientRequestId para evitar duplicados.
- Events: diseñar contratos de eventos idempotentes y usar DLQ para reintentos.

Desarrollo local (rápido)
-------------------------
- Abrir carpeta del servicio y seguir su README para instalar dependencias:
  - cd microservicios/<servicio>
  - npm install
  - cp .env.example .env (ajustar valores)
  - npm run dev
- Recomendar docker-compose para entornos locales (Postgres, Redis, Broker). Crear docker-compose.yml en raíz si se desea levantar todos los servicios y dependencias juntos.

Ejemplo de comandos útiles
--------------------------
- Instalar deps de un servicio:
  - cd microservicios/usuarios
  - npm install
- Levantar un servicio en dev:
  - npm run dev (usar nodemon)
- Probar:
  - npm test
- Construir imagen Docker:
  - docker build -t <servicio>:dev .
- Ejecutar imagen:
  - docker run --env-file .env -p <PORT>:<PORT> <servicio>:dev

CI / CD (recomendado)
---------------------
- Pipeline por servicio: lint -> tests -> build image -> push image -> despliegue.
- Ejecutar migraciones controladas durante despliegue.
- Contract tests entre productores/consumidores de eventos antes de promover.

Infra y despliegue
------------------
- Orquestador: Kubernetes / Docker Compose / ECS según preferencia.
- Secrets: usar vault/secret manager del proveedor.
- Monitoreo y alertas: Prometheus + Alertmanager / Grafana.
- Storage de assets: S3-compatible (o provider cloud).

Documentación y contratos
-------------------------
- Mantener OpenAPI/Swagger básico por servicio para rutas públicas.
- Documentar events (topic, payload, version) en README de cada servicio.
- Registrar cambios de contrato en CHANGELOG.md de cada microservicio.

Checklist inicial al crear un nuevo microservicio
-------------------------------------------------
- [ ] Crear carpeta microservicios/<nombre>
- [ ] Añadir README.md siguiendo plantilla del monorepo
- [ ] package.json con scripts: dev, test, lint, start
- [ ] .env.example con variables mínimas
- [ ] Dockerfile y (opcional) docker-compose service entry
- [ ] Implementar GET /health
- [ ] Logging, error handler y middleware de auth básico
- [ ] Migrations y seeders iniciales
- [ ] Tests unitarios y de integración mínimos

Referencias
-----------
Consultar los README.md individuales dentro de cada microservicio para detalles de modelos, endpoints y checklist específicos. Mantener coherencia con estas convenciones para facilitar operaciones, testing y despliegue.