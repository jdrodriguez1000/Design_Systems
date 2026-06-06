# Stack Tecnológico — MeetingApp

## Stack recomendado por capa

---

### Frontend / Cliente

**Recomendación: React + TypeScript + Vite**

| Opción | Para este proyecto |
|--------|-------------------|
| **React + TypeScript** | ✅ Ecosistema maduro, TypeScript detecta errores de tipo en tiempo de compilación — crítico cuando el frontend maneja roles y estados de reserva |
| **Vue 3** | ✅ Igualmente válido — Composition API es ergonómica para este caso de uso |
| **Vanilla JS** | ❌ Para una app con múltiples vistas, autenticación, y estado complejo (reservas, notificaciones, roles), un framework evita una cantidad significativa de código manual |
| **Next.js** | ❌ SSR/SSG añade complejidad sin beneficio para una app interna con auth |

**Librerías clave**:
- `react-query` o `SWR`: gestión de requests asíncronos y polling de notificaciones
- `react-hook-form`: formulario de reserva con validación
- `date-fns`: manipulación de fechas sin el peso de Moment.js

---

### Backend / API

**Recomendación: FastAPI (Python) o Express (Node.js/TypeScript)**

| Opción | Justificación |
|--------|--------------|
| **FastAPI (Python)** | ✅ Tipado nativo con Pydantic, generación automática de docs OpenAPI, async/await nativo, ecosistema Python maduro para lógica de negocio compleja |
| **Express + TypeScript** | ✅ Si el equipo ya conoce JavaScript, mantiene un solo lenguaje en toda la stack |
| **Django** | Válido pero excesivo — Django admin, ORM propio, y estructura MVC son más de lo necesario para una API REST |
| **Spring Boot (Java)** | Overkill en complejidad de configuración para este tamaño |

**Elección final si el equipo es nuevo**: FastAPI. La generación automática de documentación OpenAPI permite probar los endpoints directamente desde el browser — muy útil durante el desarrollo.

**Librerías clave** (FastAPI):
- `SQLAlchemy 2.0`: ORM con soporte async nativo
- `alembic`: migraciones de BD versionadas
- `python-jose`: generación y verificación de JWT
- `bcrypt`: hashing de contraseñas
- `pydantic`: validación de requests/responses

---

### Base de datos principal

**PostgreSQL 16**

Ya documentado en `010_database_model.md`. Justificación clave: ACID, soporte nativo de `SELECT FOR UPDATE`, queries de solapamiento de intervalos en SQL nativo, y el tipo `TIMESTAMPTZ` para timestamps con zona horaria.

---

### Base de datos secundaria (caché)

**No aplica en V1.** Si se añaden notificaciones en tiempo real (DF-02) o el sistema crece:
- **Redis**: para caché de disponibilidad de salas o como pub/sub para SSE/WebSockets

---

### Infraestructura y cloud

**Opción recomendada: Railway o Render (PaaS simple)**

| Opción | Costo | Complejidad | Para este proyecto |
|--------|-------|-------------|-------------------|
| **Railway** | ~$5-20/mes | Baja | ✅ Deploy de Docker + PostgreSQL managed en una plataforma |
| **Render** | ~$7-25/mes | Baja | ✅ Mismo perfil que Railway |
| **AWS EC2 + RDS** | ~$20-50/mes | Alta | ❌ Complejidad de IAM, VPC, grupos de seguridad innecesaria |
| **Fly.io** | ~$3-15/mes | Media | ✅ Más técnico pero muy eficiente para apps pequeñas |
| **VPS propio (DigitalOcean, Hetzner)** | ~$5-10/mes | Media | ✅ Máximo control, requiere gestión de updates del SO |

**Recomendación**: Railway o Render para no distraer atención de la lógica de negocio con infraestructura.

---

### Herramientas de observabilidad

| Herramienta | Propósito | Alternativa |
|-------------|----------|------------|
| **Logs estructurados JSON** (integrado) | Base de toda la observabilidad | — |
| **Sentry (free tier)** | Alertas de excepciones no manejadas | Rollbar |
| **Railway/Render built-in metrics** | CPU, memoria, latencia | — |
| **pgAdmin o TablePlus** | Exploración de la BD en desarrollo | — |

No se necesita Datadog ni Grafana para esta escala.

---

## Compatibilidad del stack

```
React (TypeScript) + Vite
    │ HTTPS / REST API (JSON)
FastAPI (Python) + SQLAlchemy
    │ SQL (PostgreSQL driver - asyncpg)
PostgreSQL 16
    │
Railway/Render (hosting)
    + GitHub Actions (CI/CD)
    + Sentry (error tracking)
```

Todo se comunica con contratos claros: OpenAPI spec entre frontend y backend, SQL entre backend y DB. Si se reemplaza cualquier capa, el contrato permanece.

---

## Alternativas descartadas con razón específica

| Tecnología | Por qué se descartó para este proyecto |
|-----------|---------------------------------------|
| **Firebase Realtime DB** | Sin soporte para transacciones ACID del tipo que requiere el sistema de desplazamiento |
| **MongoDB** | Queries de solapamiento de intervalos son más naturales en SQL; sin ventajas para este modelo de datos |
| **GraphQL** | 9 endpoints bien definidos no necesitan la flexibilidad de GraphQL; añade complejidad de schema sin valor |
| **Kubernetes** | Overkill para un solo servidor con una app de empresa pequeña |
| **Microservicios** | 3 dominios (reservas, usuarios, notificaciones) en un monolito bien estructurado son suficientes para esta escala |

---

## Curva de aprendizaje

| Tecnología | Curva | Tiempo estimado para alguien nuevo |
|-----------|-------|-----------------------------------|
| FastAPI | Baja-Media | 1 semana para tener una API funcional |
| SQLAlchemy 2.0 | Media | 2 semanas para dominar ORM + migraciones |
| React + TypeScript | Media | 2-3 semanas si viene de JavaScript |
| PostgreSQL | Media | 1-2 semanas para queries básicas y transacciones |
| Docker Compose | Baja | 2-3 días para setup de desarrollo |

El stack completo es dominable por un desarrollador en 4-6 semanas — apropiado para un equipo pequeño o un solo desarrollador.
