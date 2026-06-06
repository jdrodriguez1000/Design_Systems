# Protocolos de Comunicación — InventariApp

## Comunicación sincrónica: REST API sobre HTTPS

**Recomendación: REST sobre JSON/HTTPS**

La decisión es clara. El sistema tiene operaciones bien definidas y un modelo de datos estable. No hay necesidad de la flexibilidad de GraphQL ni del rendimiento de gRPC.

### ¿Por qué REST y no GraphQL?

GraphQL brilla cuando el frontend necesita consultar relaciones complejas de datos con formas variables (GitHub API, Shopify Storefront). Para InventariApp, las operaciones son CRUD bien definidas con un conjunto fijo de vistas:

| Operación | Endpoint REST |
|-----------|--------------|
| Listar productos con stock calculado | `GET /productos?estado=activo` |
| Registrar entrada | `POST /movimientos` |
| Registrar salida | `POST /movimientos` |
| Registrar contra-movimiento | `POST /movimientos` |
| Ver historial de un producto | `GET /productos/:id/movimientos` |
| Listar alertas activas | `GET /alertas?estado=activa` |
| Ver detalle de alerta con contexto 30 días | `GET /alertas/:id/contexto` |
| Configurar stock mínimo | `PATCH /productos/:id/stock-minimo` |
| Historial de excepciones | `GET /movimientos?excepcion=true` |
| Crear orden de reposición | `POST /ordenes` |
| Confirmar orden | `POST /ordenes/:id/confirmar` |
| Marcar orden como recibida | `POST /ordenes/:id/recibir` |
| KPIs del panel ejecutivo | `GET /dashboard/kpis` |
| Tabla de productos del panel | `GET /dashboard/productos?categoria=X&estado=Y` |

14 endpoints claros. GraphQL añadiría un schema y resolvers por campo sin ningún beneficio.

### ¿Por qué REST y no gRPC?

gRPC es para comunicación entre microservicios internos de alta frecuencia. Los browsers no soportan gRPC nativo. Para una app web estándar, REST es la opción sin fricciones.

---

## Comunicación en tiempo real: polling simple

**No se necesita WebSockets ni SSE en v1.**

Las alertas de stock bajo son el único "dato en tiempo real" del sistema. La pregunta es: ¿cómo se entera Diana de que hay una nueva alerta?

### Opción A: Polling (recomendado para v1)

El cliente refresca el panel o la vista de alertas al hacer clic o al navegar. Adicionalmente, un polling automático de `GET /alertas?estado=activa` cada 60 segundos en la pantalla principal es suficiente.

```
Cliente ──[GET /alertas?estado=activa]──▶ Servidor ──▶ DB
         ◀──[{alertas: [{...}]}]──
         ... 60 segundos después ...
         ──[GET /alertas?estado=activa]──▶ Servidor ──▶ DB
```

**¿Por qué es suficiente?**: las alertas de stock bajo no son time-critical al nivel de milisegundos. Si Andrés despacha el último cartucho a las 14:30 y Diana se entera a las 14:31, el impacto operativo es cero — la orden de reposición tardará días en llegar de todas formas.

### Opción B: Server-Sent Events (SSE) — para v2 si el polling resulta insatisfactorio

Si la empresa crece o si los stakeholders reportan que el delay de 60s causa fricción, SSE permite notificaciones inmediatas con menor complejidad que WebSockets:

```
Cliente ──[GET /events]──▶ Servidor
         ◀──[data: {tipo: 'nueva_alerta', producto_id: '...'}]──  (push inmediato)
```

SSE es unidireccional (servidor → cliente) — suficiente para notificaciones de alertas.

### Opción C: WebSockets — no aplica

La bidireccionalidad de WebSockets no se necesita. Las alertas solo fluyen del servidor al cliente.

---

## Diseño de la API REST

### Principios clave

**1. Sin autenticación en v1** (exclusión confirmada con stakeholders)

No hay header `Authorization`. Todos los endpoints son públicos dentro de la red interna. Esto es un riesgo aceptado documentado (R-01 en scope_boundaries.md).

**2. Respuestas de error consistentes**

```json
// 409 Conflict — salida bloqueada por stock insuficiente
{
  "error": "INSUFFICIENT_STOCK",
  "message": "Stock insuficiente. Stock actual disponible: 5 unidades.",
  "stock_actual": 5,
  "puede_forzar": true
}

// 422 Validation Error — fecha futura
{
  "error": "INVALID_DATE",
  "message": "La fecha/hora del evento no puede ser futura."
}

// 400 Bad Request — intento de entrada a producto discontinuado
{
  "error": "PRODUCT_DISCONTINUED",
  "message": "Este producto está discontinuado y no puede recibir nuevas entradas."
}
```

**3. Idempotencia para operaciones críticas**

El registro de movimientos debe ser idempotente ante retries del cliente (ej: doble clic, reconexión). Implementar con un `idempotency_key` en el body o header: si el servidor ya procesó un movimiento con esa clave, devuelve el movimiento ya creado sin insertar uno nuevo.

**4. Timestamps en ISO 8601 con zona horaria UTC**

```json
{
  "fecha_hora_evento": "2026-06-06T14:30:00Z",
  "timestamp_creacion": "2026-06-06T14:31:15Z"
}
```

La conversión a hora local la hace el frontend. Esto evita bugs de DST en el servidor.

---

## Formato de datos: JSON

JSON es la elección correcta. Protocol Buffers añaden complejidad de compilación de schemas sin beneficio para este volumen. Avro está diseñado para streaming de datos (Kafka) — no aplica aquí.

---

## Comunicación entre componentes internos

El monolito no tiene comunicación entre servicios en el sentido de llamadas HTTP. La comunicación interna es:
- **Llamadas directas** (función a función) dentro del mismo proceso
- **Event bus in-process** para el patrón Observer (AlertEvaluator escucha eventos de MovementService)
- **Transacciones de BD** para garantizar atomicidad del trio movimiento + stock + alerta

---

## Trade-offs

| Opción | Latencia | Complejidad | Elección |
|--------|---------|-------------|---------|
| REST + polling 60s | ~0-60s para alertas | Muy baja | ✅ v1 |
| REST + SSE | ~0s para alertas | Baja-Media | Para v2 si se necesita |
| REST + WebSockets | ~0s para alertas | Media | Innecesario (unidireccional) |
| GraphQL | Sin cambio | Alta | No hay beneficio |
