# Servicio: Web Clientes (Página Pública)

Descripción
------------
Aplicación cliente pública destinada a compradores y visitantes. Provee catálogo, búsqueda, carrito, checkout, seguimiento de pedidos, cuentas de usuario y contenido público (landing, FAQ). Se integra con microservicios backend a través del API Gateway.

Objetivos
---------
- Experiencia de compra fluida y accesible en desktop y móvil.
- Autenticación de clientes, gestión de cuentas y órdenes.
- Búsqueda y filtrado del catálogo con rendimiento.
- Checkout seguro e integración con pasarelas de pago.
- SEO y rendimiento (SSR/SSG cuando aplique).
- Observabilidad, tests y despliegue independiente.

Stack recomendado
-----------------
- Frontend: React + Vite con TypeScript
- UI: TailwindCSS / Chakra UI / MUI
- State: React Query / SWR / Zustand
- Payment: integrar pasarelas (Stripe, PayU, etc.) mediante backend
- Search: ElasticSearch/OpenSearch o API de productos
- CI: Vercel/Netlify/Cloud Run / Docker + NGINX
- Tests: Jest + React Testing Library, E2E con Playwright/Cypress

Estructura de carpetas (sugerida)
---------------------------------
/microservicios/web-clientes/
- frontend/
  - src/
    - pages/ o app/
    - components/
    - hooks/
    - services/ (API via API Gateway)
    - styles/
    - public/
  - package.json
- infra/ (nginx.conf, Dockerfile.prod)
- tests/ (e2e / integración)
- .env.example
- README.md
- Dockerfile.frontend

Variables de entorno (.env.example)
----------------------------------
NEXT_PUBLIC_API_BASE_URL=https://api.example.com/v1
NEXT_PUBLIC_APP_NAME=TiendaOnline
NEXT_PUBLIC_SEARCH_URL=
NEXT_PUBLIC_STRIPE_PK=
NEXT_PUBLIC_FEATURE_FLAGS_URL=
NODE_ENV=production
PORT=3012

Autenticación y cuentas
-----------------------
- Soporte para login/signup con email/password y social logins (OIDC/OAuth).
- Tokens manejados con cookies HttpOnly (recomendado) o storage según política.
- Rutas protegidas: /mi-cuenta, /pedidos, /checkout.
- Flujos de recuperación de contraseña y verificación de email.

Integración con microservicios
------------------------------
- Todas las llamadas al backend deben ir por API Gateway (NEXT_PUBLIC_API_BASE_URL).
- Usar correlationId/requestId en headers.
- Checkout: orquestar con Pedidos, Almacén, Pagos y Notificaciones.
- Suscripción a actualizaciones en tiempo real (SSE/WebSocket) para estado de envío (opcional).

Rutas y páginas mínimas
-----------------------
- / (landing / promociones)
- /productos, /productos/:slug
- /categorias/:slug
- /busqueda?q=
- /producto/:id (detalle)
- /carrito
- /checkout
- /mi-cuenta (perfil, direcciones)
- /mis-pedidos/:id (seguimiento)
- /ayuda / FAQ / contacto
- /auth/login, /auth/register, /auth/forgot-password

SEO, rendimiento y accesibilidad
--------------------------------
- Usar SSR/SSG para páginas públicas críticas (landing, producto).
- Meta tags y OpenGraph dinámicos.
- Optimizar imágenes (next/image o similar), lazy loading y caching.
- Lighthouse >=90 como objetivo en páginas claves.
- Accesibilidad (a11y) básica: semántica, labels, focus management.

Checkout y pagos
----------------
- No exponer claves secretas en el cliente.
- Realizar la integración de pagos a través del backend para seguridad y firma.
- Manejar idempotencia y reconcilición de pagos (clientRequestId).
- Mostrar estados claros al usuario (pagado, pendiente, fallido).

Seguridad
---------
- Cookies HttpOnly + SameSite=strict recomendadas.
- CSRF protection para POST sensibles si no se usa cookies same-site.
- Rate limiting por IP en endpoints públicos del Gateway.
- Sanitizar entradas y validar en frontend antes de enviar al backend.
- Evitar almacenar información de pago sensible en el cliente.

Testing
-------
- Unit tests: componentes y hooks con Jest + React Testing Library.
- Integration/E2E: Playwright/Cypress para flujo crítico (signup, búsqueda, checkout).
- Tests de rendimiento y SEO (audits automatizados).
- Tests de contract con API Gateway (mocks / schemas).

Build & despliegue
------------------
- Comandos típicos:
  - npm install
  - npm run dev
  - npm run build
  - npm run start
- Dockerfile.frontend (ejemplo):
  - FROM node:20, build, copiar build a NGINX, servir estáticos
- Recomendación: desplegar en CDN o plataforma con edge (Vercel, Netlify o Cloud Run + CDN).
- Integrar CDN para assets estáticos y cacheo.

Observabilidad
--------------
- Exponer GET /health
- Integrar RUM/monitoring (Sentry, Datadog RUM) para errores y performance en cliente.
- Logs estructurados desde backend; correlacionar con requestId enviado desde frontend.
- Métricas: tasa de conversión, abandono de carrito, tiempos de checkout.

Privacidad y cumplimiento
-------------------------
- Cumplir regulaciones locales (protección de datos, Cookies consent).
- Consentimiento explícito para tracking/analytics.
- Opciones para borrar / exportar datos de usuario si aplica.

Internacionalización y localización
-----------------------------------
- Soportar i18n y formatos locales (moneda, fechas).
- Detectar locale por dominio/header y permitir cambio manual.

Checklist de implementación
---------------------------
- [ ] Inicializar proyecto (Next.js/React + TypeScript)
- [ ] Diseño mobile-first y sistema de componentes
- [ ] Integrar llamadas al API Gateway con manejo de errores y retries
- [ ] Implementar búsqueda, catálogo, carrito y checkout
- [ ] Autenticación y gestión de cuentas (email, social)
- [ ] Tests unitarios y E2E para flujos críticos
- [ ] Dockerfile.frontend y configuración de despliegue + CDN
- [ ] Healthcheck, RUM y monitoring (Sentry)
- [ ] Política de cookies y cumplimiento legal

Comandos útiles
---------------
- npm install
- npm run dev
- npm run build
- npm run start
- npx playwright test  (o npx cypress open)

Referencias
-----------
Seguir convenciones del monorepo: usar API Gateway como único punto de entrada al backend. Documentar cualquier excepción en este README.