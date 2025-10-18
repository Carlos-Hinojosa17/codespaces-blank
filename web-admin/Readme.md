# Servicio: Web Admin (Sistema Web - Interno)

Descripción
------------
Aplicación web interna usada por trabajadores (administradores, vendedores, almacenistas, etc.). Provee UI (React/Next) y API backend para funcionalidades administrativas: gestión de usuarios, productos, ventas, pedidos, almacenes, reportes y configuración. Diseñado para integrarse con los microservicios del backend a través del API Gateway.

Objetivos
---------
- Interfaz responsiva y segura para usuarios internos.
- Autenticación/SSO y autorización por roles.
- Consumir APIs de microservicios (usuarios, productos, ventas, pedidos, almacen, reportes, notificaciones, configuración) mediante API Gateway.
- Dashboards, formularios CRUD, acciones en lote y herramientas operativas.
- Buenas prácticas: testing, accesibilidad básica, observabilidad y despliegue independiente.

Stack recomendado
-----------------
- Frontend: React (18+) o Next.js (SSR/SSG) con TypeScript
- UI: TailwindCSS / Chakra UI / MUI
- State: React Query / SWR / Zustand
- Backend (opcional para server-side needs): Node.js 20+ + Express (API proxy, SSR functions)
- Autenticación: JWT / OIDC (Keycloak/Auth0) para SSO
- CI: Vite/Next build, tests con Jest + React Testing Library, E2E con Playwright/Cypress
- Docker, NGINX para servir estáticos en producción

Estructura de carpetas (sugerida)
---------------------------------
/microservicios/web-admin/
- frontend/ (Next.js or React app)
  - src/
    - pages/ o app/
    - components/
    - hooks/
    - services/ (llamadas a API via API Gateway)
    - styles/
  - public/
  - package.json
- backend/ (opcional - proxy/SSR/API helpers)
  - src/
    - config/
    - routes/
    - middleware/
    - utils/
  - package.json
- infra/ (nginx.conf, Dockerfile.prod, docker-compose.yml)
- tests/ (integration / e2e)
- .env.example
- README.md
- Dockerfile (frontend) / Dockerfile.backend (si aplica)

Variables de entorno (.env.example)
----------------------------------
NEXT_PUBLIC_API_BASE_URL=https://api.example.com/v1
NEXT_PUBLIC_APP_NAME=SistemaWeb
AUTH_ISSUER=https://auth.example.com
AUTH_CLIENT_ID=web-admin-client
JWT_AUDIENCE=
NODE_ENV=development
PORT_FRONTEND=3009
PORT_BACKEND=3011

Autenticación y autorización
-----------------------------
- SSO/OIDC recomendado (Keycloak / Auth0) para inicio de sesión centralizado.
- JWT en navegador (cookie HttpOnly o storage según recomendación de seguridad).
- Middleware de protección de rutas en frontend (roles: admin, ventas, almacen, reportes).
- Controles de UI basados en permisos (feature flags desde microservicio de configuración).

Integración con microservicios
------------------------------
- Todas las llamadas a microservicios pasar por API Gateway (NEXT_PUBLIC_API_BASE_URL).
- Usar correlationId/requestId en headers para trazabilidad.
- Suscribirse a eventos relevantes (via WebSocket o SSE) para actualizaciones en tiempo real (p. ej. cambios de inventario, estado de pedidos).

Rutas y páginas mínimas (frontend)
----------------------------------
- /login
- /logout
- /dashboard
- /usuarios (list, create, edit)
- /productos (catálogo, variantes, import)
- /ventas (list, detalle, notas)
- /pedidos (list, detalle)
- /almacen (stock, reservas, movimientos)
- /reportes (generar, descargar, schedule)
- /configuracion (feature flags, assets)
- /notificaciones (colas, templates)

Buenas prácticas UX / Seguridad
------------------------------
- Forzar MFA para roles administrativos.
- Cookies HttpOnly + SameSite=strict para tokens; CSRF protection en formularios si aplica.
- Validación y sanitización de todos los inputs antes de enviarlos a backend.
- Evitar exponer secretos en el cliente.
- Accesibilidad básica (a11y) y tests automatizados para componentes críticos.

Testing
-------
- Unit tests: Jest + React Testing Library para componentes y hooks.
- Integration/E2E: Playwright o Cypress para flujos críticos (login, creación de pedido/venta).
- Tests de contrato: validar integraciones con API Gateway/microservicios (mockar respuestas).

Build & despliegue
------------------
- Frontend:
  - npm install
  - npm run build
  - npm run start (Next.js) o servir estáticos con NGINX
- Backend (si existe):
  - npm install
  - npm run build
  - npm start
- Docker:
  - docker build -t web-admin-frontend:latest -f Dockerfile.frontend .
  - docker run --env-file .env -p 3009:3009 web-admin-frontend:latest

Observabilidad
--------------
- Exponer /health en frontend/backend.
- Logging estructurado (pino/winston) y forward a agregador.
- Métricas básicas (P95 latencia, errores) y tracing (OpenTelemetry) opcional.
- Monitorizar errores frontend con Sentry o similar.

Checklist de implementación
---------------------------
- [ ] Inicializar proyecto frontend (Next.js/React + TypeScript)
- [ ] Integrar autenticación (OIDC/SSO) y protección de rutas
- [ ] Implementar layout, header/nav y gestión de permisos
- [ ] Consumir APIs a través de API Gateway con manejo de errores y retries
- [ ] Pages: usuarios, productos, ventas, pedidos, almacen, reportes, configuracion, notificaciones
- [ ] Tests unitarios e E2E para flujos críticos
- [ ] Dockerfile y configuración de despliegue (NGINX/Ingress)
- [ ] Healthcheck, logging y métricas básicas
- [ ] Documentación de uso para equipo interno

Comandos útiles
---------------
- Desde frontend:
  - npm install
  - npm run dev
  - npm run build
  - npm run start
- E2E:
  - npx playwright test  (o npx cypress open)

Referencias
-----------
Seguir convenciones del monorepo: usar API Gateway como único punto de entrada a microservicios y documentar cualquier excepción en este README.
```// filepath: /microservicios/web-admin/README.md

# Servicio: Web Admin (Sistema Web - Interno)

Descripción
------------
Aplicación web interna usada por trabajadores (administradores, vendedores, almacenistas, etc.). Provee UI (React/Next) y API backend para funcionalidades administrativas: gestión de usuarios, productos, ventas, pedidos, almacenes, reportes y configuración. Diseñado para integrarse con los microservicios del backend a través del API Gateway.

Objetivos
---------
- Interfaz responsiva y segura para usuarios internos.
- Autenticación/SSO y autorización por roles.
- Consumir APIs de microservicios (usuarios, productos, ventas, pedidos, almacen, reportes, notificaciones, configuración) mediante API Gateway.
- Dashboards, formularios CRUD, acciones en lote y herramientas operativas.
- Buenas prácticas: testing, accesibilidad básica, observabilidad y despliegue independiente.

Stack recomendado
-----------------
- Frontend: React (18+) o Next.js (SSR/SSG) con TypeScript
- UI: TailwindCSS / Chakra UI / MUI
- State: React Query / SWR / Zustand
- Backend (opcional para server-side needs): Node.js 20+ + Express (API proxy, SSR functions)
- Autenticación: JWT / OIDC (Keycloak/Auth0) para SSO
- CI: Vite/Next build, tests con Jest + React Testing Library, E2E con Playwright/Cypress
- Docker, NGINX para servir estáticos en producción

Estructura de carpetas (sugerida)
---------------------------------
/microservicios/web-admin/
- frontend/ (Next.js or React app)
  - src/
    - pages/ o app/
    - components/
    - hooks/
    - services/ (llamadas a API via API Gateway)
    - styles/
  - public/
  - package.json
- backend/ (opcional - proxy/SSR/API helpers)
  - src/
    - config/
    - routes/
    - middleware/
    - utils/
  - package.json
- infra/ (nginx.conf, Dockerfile.prod, docker-compose.yml)
- tests/ (integration / e2e)
- .env.example
- README.md
- Dockerfile (frontend) / Dockerfile.backend (si aplica)

Variables de entorno (.env.example)
----------------------------------
NEXT_PUBLIC_API_BASE_URL=https://api.example.com/v1
NEXT_PUBLIC_APP_NAME=SistemaWeb
AUTH_ISSUER=https://auth.example.com
AUTH_CLIENT_ID=web-admin-client
JWT_AUDIENCE=
NODE_ENV=development
PORT_FRONTEND=3009
PORT_BACKEND=3011

Autenticación y autorización
-----------------------------
- SSO/OIDC recomendado (Keycloak / Auth0) para inicio de sesión centralizado.
- JWT en navegador (cookie HttpOnly o storage según recomendación de seguridad).
- Middleware de protección de rutas en frontend (roles: admin, ventas, almacen, reportes).
- Controles de UI basados en permisos (feature flags desde microservicio de configuración).

Integración con microservicios
------------------------------
- Todas las llamadas a microservicios pasar por API Gateway (NEXT_PUBLIC_API_BASE_URL).
- Usar correlationId/requestId en headers para trazabilidad.
- Suscribirse a eventos relevantes (via WebSocket o SSE) para actualizaciones en tiempo real (p. ej. cambios de inventario, estado de pedidos).

Rutas y páginas mínimas (frontend)
----------------------------------
- /login
- /logout
- /dashboard
- /usuarios (list, create, edit)
- /productos (catálogo, variantes, import)
- /ventas (list, detalle, notas)
- /pedidos (list, detalle)
- /almacen (stock, reservas, movimientos)
- /reportes (generar, descargar, schedule)
- /configuracion (feature flags, assets)
- /notificaciones (colas, templates)

Buenas prácticas UX / Seguridad
------------------------------
- Forzar MFA para roles administrativos.
- Cookies HttpOnly + SameSite=strict para tokens; CSRF protection en formularios si aplica.
- Validación y sanitización de todos los inputs antes de enviarlos a backend.
- Evitar exponer secretos en el cliente.
- Accesibilidad básica (a11y) y tests automatizados para componentes críticos.

Testing
-------
- Unit tests: Jest + React Testing Library para componentes y hooks.
- Integration/E2E: Playwright o Cypress para flujos críticos (login, creación de pedido/venta).
- Tests de contrato: validar integraciones con API Gateway/microservicios (mockar respuestas).

Build & despliegue
------------------
- Frontend:
  - npm install
  - npm run build
  - npm run start (Next.js) o servir estáticos con NGINX
- Backend (si existe):
  - npm install
  - npm run build
  - npm start
- Docker:
  - docker build -t web-admin-frontend:latest -f Dockerfile.frontend .
  - docker run --env-file .env -p 3009:3009 web-admin-frontend:latest

Observabilidad
--------------
- Exponer /health en frontend/backend.
- Logging estructurado (pino/winston) y forward a agregador.
- Métricas básicas (P95 latencia, errores) y tracing (OpenTelemetry) opcional.
- Monitorizar errores frontend con Sentry o similar.

Checklist de implementación
---------------------------
- [ ] Inicializar proyecto frontend (Next.js/React + TypeScript)
- [ ] Integrar autenticación (OIDC/SSO) y protección de rutas
- [ ] Implementar layout, header/nav y gestión de permisos
- [ ] Consumir APIs a través de API Gateway con manejo de errores y retries
- [ ] Pages: usuarios, productos, ventas, pedidos, almacen, reportes, configuracion, notificaciones
- [ ] Tests unitarios e E2E para flujos críticos
- [ ] Dockerfile y configuración de despliegue (NGINX/Ingress)
- [ ] Healthcheck, logging y métricas básicas
- [ ] Documentación de uso para equipo interno

Comandos útiles
---------------
- Desde frontend:
  - npm install
  - npm run dev
  - npm run build
  - npm run start
- E2E:
  - npx playwright test  (o npx cypress open)

Referencias
-----------
Seguir convenciones del monorepo: usar API Gateway como único punto de entrada a microservicios y documentar cualquier excepción en este README.