# Microservicio: API Gateway

Descripción
------------
Gateway que expone la API pública y coordina tráfico entre clientes y microservicios internos. Se encarga de enrutar, autenticar, autorizar, aplicar políticas de seguridad, caching, rate limiting, logging y tracing. Actúa como single entry point para la plataforma.

Objetivos
---------
- Enrutar solicitudes a microservicios con prefijos (/usuarios, /productos, /ventas, /pedidos, /almacen, /reportes, /configuracion, /notificaciones).
- Validar y propagar autenticación (JWT) y claims.
- Implementar políticas cross-cutting: rate limiting, CORS, Caching, circuit breaker, timeouts.
- Centralizar transformación mínima de requests/responses (headers, versiones).
- Proveer métricas, logs estructurados y tracing para observabilidad.
- Manejar TLS/terminación (o delegar al proxy inverso/ingress).

Stack recomendado
-----------------
- Node.js 20+ (Express/Koa/Fastify) o soluciones dedicadas (Kong, Traefik, NGINX, Envoy).
- Redis para rate limiting y cacheo.
- JWT verification (jwks-rsa si se usa JWKs/OIDC).
- Circuit breaker: opossum or built-in client libs.
- OpenTelemetry para tracing.
- Prometheus + Grafana para métricas.
- Docker, Kubernetes Ingress Controller (si aplica).

Estructura de carpetas (base)
-----------------------------
/microservicios/api-gateway/
- src/
  - config/ (env.js, servicesMap.js, redis.js)
  - middleware/ (auth.js, rateLimit.js, cors.js, cache.js, timeout.js, errorHandler.js)
  - routes/ (proxyRoutes.js)
  - utils/ (logger.js, tracer.js, health.js)
  - server.js / app.js
- tests/
- package.json
- .env
- Dockerfile
- README.md

Variables de entorno (.env)
--------------------------
Ejemplo mínimo:
PORT=3000
NODE_ENV=production
JWT_JWKS_URI=                 # si se usa JWKS/OIDC
JWT_AUDIENCE=
JWT_ISSUER=
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100
REDIS_URL=redis://localhost:6379
SERVICE_DISCOVERY_URL=        # opcional: consul/etcd
TIMEOUT_MS=15000
TRACING_ENDPOINT=
CORS_ORIGINS=*

Responsabilidades clave
-----------------------
- Enrutamiento: mapear prefijos y rutas a servicios internos; soporte para rewriting y versioning.
- Autenticación: validar JWT y opcionalmente delegar a auth service; inyectar claims en headers internos.
- Autorización: aplicar reglas simples de ACL por ruta/role o delegar a microservicio de autorización.
- Rate limiting: por IP, por API key o por usuario, con backend Redis.
- Caching: cachear respuestas GET seguras con TTL y invalidación por eventos.
- Timeouts y retries: configurar timeouts por ruta y retries idempotentes con backoff.
- Circuit breaker: evitar cascadas cuando un downstream falla.
- Observabilidad: incluir correlationId, logs estructurados, métricas de latencia/errores, y tracing distribuido.
- Seguridad: aplicar headers (HSTS, CSP, X-Frame-Options), limitar payload size, proteger against slowloris.

API y endpoints recomendados
----------------------------
- Proxy: rutas públicas p. ej. GET /v1/productos -> /productos/v1/...
- Administración del gateway (protegido):
  - GET /health
  - GET /metrics (Prometheus)
  - GET /routes (lista de rutas y estado)
  - POST /reload (recargar configuración sin reiniciar)
  - POST /cache/invalidate (invalidar claves)

Integración con infraestructura
-------------------------------
- Service discovery (Consul / Eureka / etcd) para resolver endpoints dinámicos.
- Load balancing round-robin o basado en pesos.
- TLS: certificados gestionados por orquestador (K8s ingress) o por el gateway.
- Integración con Identity Provider (OIDC) para login y JWKS rotation.
- Logs centralizados (ELK / Loki) y métricas a Prometheus.

Consideraciones de seguridad
----------------------------
- Rechazar solicitudes sin Authorization cuando aplique; ocultar detalles de errores.
- Validar tamaño máximo de body y tipos de contenido permitidos.
- CORS restrictivo en producción.
- Protección contra brute force y abuse patterns (WAF / rate limiting).
- Registrar y alertar patrones anómalos.

Testing y despliegue
--------------------
- Tests unitarios para middlewares y e2e con servicios simulados.
- Pruebas de carga y resiliencia (chaos tests para downstream failures).
- Desplegar con rolling updates; utilizar readiness/liveness probes si se ejecuta en K8s.
- Strategy: preferir gateway ligero + componentes especializados (Kong/Envoy) cuando la carga crece.

Comandos de desarrollo útiles
-----------------------------
- npm install
- npm run dev
- npm test
- docker build -t api-gateway:dev .
- docker run --env-file .env -p 3000:3000 api-gateway:dev

Checklist de implementación
---------------------------
- [ ] Implementar proxy básico y mapa de rutas.
- [ ] Middleware de verificación JWT y propagación de claims.
- [ ] Rate limiting con Redis.
- [ ] Caching GET responses con invalidación.
- [ ] Timeouts, retries y circuit breaker.
- [ ] Health, metrics y tracing integrados.
- [ ] Seguridad: headers, body limits y CORS.
- [ ] Tests unitarios e integrados.
- [ ] Dockerfile, Helm chart / K8s ingress (opcional).
- [ ] Documentar rutas y contratos en OpenAPI/Swagger.

Decisión de herramienta
-----------------------
- Para prototipos o control fino: implementar en Node (Express/Fastify).  
- Para producción con alto tráfico: evaluar gateways gestionados o proxies de alto rendimiento (Envoy, Kong, Traefik) y usar Node sólo para lógica adicional que no pueda delegar.

Referencias
-----------
Mantener coherencia con convenciones del monorepo. Documentar cualquier excepción o configuración especial en este README.