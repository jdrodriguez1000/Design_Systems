# Stack Tecnológico — InventariApp

## Stack recomendado por capa

---

### Frontend / Cliente

**Recomendación: React + TypeScript + Vite**

| Opción | Para este proyecto |
|--------|-------------------|
| **React + TypeScript** | ✅ Ecosistema maduro. TypeScript detecta errores en tiempo de compilación — crítico cuando el frontend maneja estados complejos como alertas activas, estados de órdenes y movimientos inmutables |
| **Vue 3** | ✅ Igualmente válido — Composition API es ergonómica. Si el equipo ya conoce Vue, es una elección correcta |
| **Vanilla JS** | ❌ Para una app con múltiples vistas, filtros, estados de alertas y panel KPI, la gestión del DOM manual genera código difícil de mantener |
| **Next.js** | ❌ SSR no tiene beneficio para una app interna sin SEO. Añade complejidad de servidor sin valor |

**Librerías clave**:
- `react-query` o `SWR`: gestión de requests asíncronos, caché de datos, y polling de alertas cada 60s
- `react-hook-form`: formulario de registro de movimientos con validación (fecha no futura, cantidad > 0)
- `date-fns`: manipulación de fechas — el sistema tiene lógica de fechas importantes (retroactividad, período de 30 días)
- Un componente de tabla con filtros (ej: `@tanstack/react-table`) para el panel de Luis con filtrado por categoría y estado

**Nota sobre web responsive** (requisito IS-12): el diseño debe funcionar desde móvil para que Andrés registre movimientos desde la bodega. Tailwind CSS + componentes de formulario bien adaptados son suficientes — no se necesita una librería UI pesada.

---

### Backend / API

**Recomendación: FastAPI (Python)**

| Opción | Justificación |
|--------|--------------|
| **FastAPI (Python)** | ✅ Tipado nativo con Pydantic (validación automática de requests incluyendo fecha no futura), documentación OpenAPI autogenerada, async/await para el pool de conexiones a BD, excelente para lógica de dominio compleja |
| **Express + TypeScript** | ✅ Si el equipo domina JS/TS en toda la stack, mantiene un solo lenguaje. Igualmente válido |
| **Django REST Framework** | Válido pero excesivo — Django Admin, ORM propio con migraciones propias, y estructura MVC son más de lo necesario para una API REST específica |
| **Spring Boot (Java)** | Overkill en configuración y tiempo de arranque para este tamaño |

**Elección final**: FastAPI. La generación automática de docs OpenAPI (Swagger UI) permite a Andrés y Diana probar los endpoints durante el desarrollo, y la validación con Pydantic garantiza que la fecha del evento nunca sea futura sin código adicional.

**Librerías clave (FastAPI)**:
- `SQLAlchemy 2.0` con `asyncpg`: ORM con soporte async nativo para transacciones
- `alembic`: migraciones de BD versionadas
- `pydantic v2`: validación de requests y respuestas con validadores custom (ej: fecha no futura)

```python
# Ejemplo de validador custom en Pydantic
from pydantic import field_validator
from datetime import datetime

class RegisterMovementRequest(BaseModel):
    producto_id: UUID
    tipo: str
    cantidad: int = Field(gt=0)  # must be > 0
    fecha_hora_evento: datetime

    @field_validator('fecha_hora_evento')
    def no_fecha_futura(cls, v):
        if v > datetime.now():
            raise ValueError('La fecha/hora del evento no puede ser futura (RN-08)')
        return v
```

---

### Base de datos principal

**PostgreSQL 16**

Documentado en `010_database_model.md`. Los puntos clave para la elección del stack:
- `TIMESTAMPTZ` para timestamps con zona horaria
- Índices parciales para alertas activas (`WHERE estado='activa'`)
- `CHECK` constraints para integridad de negocio (cantidad > 0, tipos válidos)
- `SELECT FOR UPDATE` para evaluación concurrente de alertas
- `gen_random_uuid()` nativo

---

### Base de datos secundaria (caché)

**No aplica en v1.**

Si el panel ejecutivo se vuelve lento en v2:
- **Redis**: para caché del stock calculado de cada producto (TTL de 60s, invalidado al insertar movimientos). Con 500 productos, son 500 entradas de caché pequeñas.

---

### Infraestructura y cloud

**Opción recomendada: Railway o Render**

| Opción | Costo/mes | Complejidad | Para este proyecto |
|--------|-----------|-------------|-------------------|
| **Railway** | ~$5-20 | Baja | ✅ Docker + PostgreSQL managed en una plataforma |
| **Render** | ~$7-25 | Baja | ✅ Mismo perfil que Railway |
| **Fly.io** | ~$3-15 | Media | ✅ Más técnico pero muy eficiente |
| **VPS propio (DigitalOcean, Hetzner)** | ~$5-10 | Media | ✅ Máximo control, gestión manual del SO |
| **AWS EC2 + RDS** | ~$20-50 | Alta | ❌ IAM, VPC, grupos de seguridad innecesarios |

**Recomendación**: Railway o Render para no distraer atención de la lógica de negocio con infraestructura. Ambos incluyen PostgreSQL managed con backups automáticos — críticos para el ledger inmutable.

---

### Herramientas de observabilidad

| Herramienta | Propósito | Alternativa |
|-------------|----------|------------|
| **Logs estructurados JSON** (integrado) | Base de toda la observabilidad | — |
| **Sentry (free tier)** | Alertas de excepciones no manejadas | Rollbar |
| **Railway/Render built-in metrics** | CPU, memoria, latencia de requests | — |
| **UptimeRobot (gratis)** | Monitoreo de disponibilidad cada 5min | Better Uptime |

No se necesita Datadog ni Grafana para esta escala.

---

## Compatibilidad del stack

```
React (TypeScript) + Vite
    │ HTTPS / REST API (JSON)
FastAPI (Python) + SQLAlchemy + asyncpg
    │ SQL async (asyncpg driver)
PostgreSQL 16
    │
Railway/Render (hosting + BD managed)
    + GitHub Actions (CI/CD)
    + Sentry (error tracking)
```

---

## Alternativas descartadas con razón específica

| Tecnología | Por qué se descartó |
|-----------|---------------------|
| **Firebase Realtime DB** | Sin transacciones ACID para movimiento + alerta en un solo commit |
| **MongoDB** | El modelo de movimientos como documentos funciona, pero los índices parciales y el `SUM` de stock son más naturales en SQL |
| **GraphQL** | 14 endpoints bien definidos no necesitan la flexibilidad de GraphQL; añade complejidad de schema sin valor |
| **Kubernetes** | Overkill para un solo servidor con una app de empresa pequeña |
| **Celery + Redis** | Para job queues de notificaciones asíncronas. No aplica en v1 sin notificaciones push |

---

## Curva de aprendizaje

| Tecnología | Curva | Tiempo estimado para alguien nuevo |
|-----------|-------|-----------------------------------|
| FastAPI + Pydantic | Baja-Media | 1 semana para API funcional con validación |
| SQLAlchemy 2.0 async | Media | 2 semanas para ORM + transacciones async |
| React + TypeScript | Media | 2-3 semanas si viene de JavaScript |
| PostgreSQL | Media | 1-2 semanas para queries + índices parciales |
| Docker Compose | Baja | 2-3 días para setup de desarrollo |

El stack completo es dominable por un desarrollador en 4-6 semanas. No hay tecnologías exóticas — todas tienen comunidades grandes y documentación extensa.
