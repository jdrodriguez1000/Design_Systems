# Mantenibilidad — InventariApp

## La lógica de alertas es el núcleo frágil del sistema

A diferencia de Contador_Calorias (sin lógica compleja) y MeetingApp (conflictos de reservas), InventariApp tiene su mayor fragilidad en la evaluación automática de alertas: cuándo generarlas, cuándo resolverlas, y cuándo el stock mínimo = 0 cambia el comportamiento. Si esta lógica queda dispersa en varios lugares, un cambio de regla (ej: agregar un tercer nivel de alerta) generará bugs difíciles de rastrear.

---

## Principios SOLID aplicados

### S — Responsabilidad Única (crítico para la lógica de alertas)

Cada módulo tiene exactamente una razón para cambiar:
- `AlertEvaluator`: cambia si cambian las reglas de cuando generar/resolver alertas
- `StockCalculator`: cambia si cambia la fórmula de cálculo de stock (ej: si se agregan ajustes por lote)
- `OverridePolicy`: cambia si cambia la política de qué hacer cuando el stock es insuficiente
- `MovementService`: cambia si cambian los tipos de movimiento o sus reglas de validación
- `OrderService`: cambia si cambia el ciclo de vida de las órdenes de reposición

**Señal de alerta**: si añadir un nuevo estado de alerta (ej: "alerta crítica" para stock = 0) requiere modificar `MovementService`, hay violación de SRP. La evaluación de alertas debe vivir en `AlertEvaluator`, no en el servicio de movimientos.

### O — Abierto/Cerrado (para futuras reglas de alerta)

Si el negocio decide añadir niveles de alerta (ej: AMARILLA cuando stock ≤ mínimo×1.5, ROJA cuando stock ≤ mínimo), la lógica de evaluación debe poder extenderse sin modificar el código existente. El pattern Observer ya garantiza esto: añadir un nuevo tipo de evaluación es añadir un nuevo listener, no modificar `MovementService`.

### D — Inversión de Dependencias (crítico para testing)

`AlertEvaluator` depende de `AlertRepository` (interfaz), no de la implementación concreta de PostgreSQL. Esto permite usar `MockAlertRepository` en tests unitarios.

---

## Separación de concerns: estructura de módulos

```
src/
├── api/
│   ├── routes/
│   │   ├── movimientos.py     # Endpoints de registro de movimientos
│   │   ├── productos.py       # Catálogo y configuración de stock mínimo
│   │   ├── alertas.py         # Consulta de alertas activas
│   │   ├── ordenes.py         # Ciclo de vida de órdenes de reposición
│   │   └── dashboard.py       # KPIs y panel de estado general
│   ├── middleware/
│   │   └── error_handler.py   # Errores consistentes en formato JSON
│   └── serializers/
│       └── schemas.py         # Pydantic schemas (request/response)
├── domain/
│   ├── movement/
│   │   ├── service.py         # MovementService — orchestración
│   │   ├── override.py        # OverridePolicy — bloquear o permitir
│   │   └── audit.py           # Lógica de flag auditoría retroactiva
│   ├── alert/
│   │   ├── evaluator.py       # AlertEvaluator — la lógica más crítica
│   │   └── service.py         # AlertService — CRUD de alertas
│   ├── stock/
│   │   └── calculator.py      # StockCalculator — SUM(movimientos)
│   └── order/
│       └── service.py         # OrderService — ciclo de vida de órdenes
├── infrastructure/
│   ├── db/
│   │   ├── repositories/      # Implementaciones concretas de los repositorios
│   │   └── migrations/        # Alembic migrations — schema versionado
│   └── events/
│       └── bus.py             # Event bus in-process (simple publish/subscribe)
└── tests/
    ├── unit/                  # Tests de dominio sin BD
    │   ├── test_alert_evaluator.py
    │   ├── test_stock_calculator.py
    │   └── test_override_policy.py
    └── integration/           # Tests con BD real (PostgreSQL en Docker)
        └── test_movement_transaction.py
```

**Regla de dependencia**: los módulos de `domain/` no importan nada de `infrastructure/` ni de `api/`. Solo conocen abstracciones (interfaces). Los módulos de `infrastructure/` implementan las abstracciones definidas en `domain/`.

---

## Modularidad: facilidad de reemplazo

| Componente | Facilidad de reemplazo | Condición |
|-----------|----------------------|-----------|
| Base de datos (PostgreSQL → MySQL) | Media | Solo cambia `infrastructure/db/` si el Repository Pattern está bien implementado |
| Política de alertas (agregar niveles) | Alta | Añadir nueva evaluación en `AlertEvaluator` sin tocar `MovementService` |
| Canal de notificaciones (in-screen → email) | Alta | Añadir `EmailNotificationListener` al event bus |
| Frontend (React → Vue) | Alta | La API REST es independiente del frontend |
| Autenticación (ninguna → JWT en v2) | Media | Añadir middleware de auth; cambiar endpoints de públicos a protegidos |

---

## Deuda técnica anticipada

| Atajo | Deuda que genera | Cuándo mitigar |
|-------|-----------------|---------------|
| Sin autenticación en v1 | Cualquier usuario puede modificar cualquier dato. Un error de Andrés puede parecer un error de Diana sin trazabilidad de quién hizo qué. | Antes de que el sistema tenga más de 3 usuarios activos o datos sensibles |
| Stock calculado sin snapshot | Con 5+ años de movimientos, la query de panel puede volverse lenta | Si latencia del panel supera 500ms, agregar campo `stock_snapshot` actualizado transaccionalmente |
| Sin paginación en historial de movimientos | Un producto con 10,000 movimientos carga la lista completa | Cuando el historial supere 500 movimientos por producto — agregar `LIMIT/OFFSET` o cursor-based pagination |
| Categorías como texto libre | Sin lista controlada de categorías, el filtro del panel puede tener inconsistencias (ej: "Papelería" vs "papeleria") | Antes de que Diana tenga 10+ categorías — crear tabla `categorias` con lista controlada |
| Cancelación de órdenes fuera de scope v1 | Órdenes en estado Confirmada de productos discontinuados quedan "huérfanas" | En v2: añadir transición `confirmada → cancelada` con motivo obligatorio |

---

## Estrategia de versionado de la API

**Versionado en URL desde v1**: `/api/v1/movimientos`

Cuando cambie un contrato que rompa clientes existentes:
1. Crear `/api/v2/movimientos` con el nuevo contrato
2. Mantener `/api/v1/movimientos` durante la transición
3. Deprecar v1 solo cuando el frontend haya migrado

---

## Code smells específicos a vigilar

| Code Smell | Señal concreta | Solución |
|-----------|---------------|---------|
| **Lógica de alerta en el endpoint** | `if nuevo_stock <= producto.stock_minimo:` dentro de `POST /movimientos` | Mover toda la evaluación a `AlertEvaluator.evaluate_after_movement()` |
| **Cálculo de stock en la vista** | `stock = sum(m.cantidad for m in movimientos if ...)` en el serializer | El cálculo pertenece a `StockCalculator`, no en la capa de presentación |
| **SQL en la capa de servicio** | `db.execute("SELECT SUM(...)...")` dentro de `MovementService` | El servicio habla con el repositorio, nunca con la BD directamente |
| **Lógica de override dispersa** | `if cantidad > stock_actual: raise...` en tres lugares diferentes | Centralizar en `OverridePolicy.check(producto, cantidad, es_override)` |
| **Histórico sin límite** | `repo.get_all_movements(producto_id)` sin límite | Añadir parámetro `limit` con default razonable desde el primer día |
