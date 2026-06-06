# Protocolos de Comunicación — MeetingApp

## A diferencia de Contador_Calorias, aquí sí hay comunicación cliente-servidor

Este sistema tiene backend propio. La comunicación entre el frontend (browser) y el backend es un diseño real que impacta la experiencia de usuario — especialmente para las notificaciones in-app.

---

## Comunicación sincrónica: REST API

**Recomendación: REST sobre JSON/HTTPS**

### ¿Por qué REST y no GraphQL?

GraphQL brilla cuando hay un frontend que necesita consultar relaciones complejas de datos con formas variables (ej: GitHub, Shopify). Para MeetingApp, las operaciones son CRUD bien definidas:

| Operación | Endpoint REST |
|-----------|--------------|
| Consultar disponibilidad | `GET /salas/disponibilidad?inicio=...&fin=...` |
| Crear reserva | `POST /reservas` |
| Cancelar reserva | `DELETE /reservas/:id` |
| Marcar como urgente | `PATCH /reservas/:id/urgente` |
| Crear usuario | `POST /usuarios` |
| Crear retroactiva | `POST /reservas/retroactiva` |
| Ver notificaciones | `GET /notificaciones` |

9 endpoints. GraphQL añadiría un schema, un resolver por campo, y complejidad de operaciones que no aportan valor aquí.

### ¿Por qué REST y no gRPC?

gRPC es ideal para comunicación entre microservicios internos de alta frecuencia. Los browsers no soportan gRPC nativo (requieren gRPC-Web + proxy). Para una app web estándar, REST es la opción sin fricciones.

---

## Comunicación en tiempo real: Polling vs. WebSockets vs. SSE

Este es el punto más interesante del sistema. Las notificaciones in-app deben ser "visibles al acceder a la aplicación" (RN-16). La pregunta es: ¿cómo se entera el cliente de que hay una notificación nueva?

### Opción A: Polling (recomendado para V1)

El cliente hace `GET /notificaciones?no_leidas=true` cada 30-60 segundos.

```
Cliente ──[GET /notificaciones]──▶ Servidor ──▶ DB
         ◀──[{notifications:[]}]──
         ... 30 segundos después ...
         ──[GET /notificaciones]──▶ Servidor ──▶ DB
         ◀──[{notifications:[{...desplazamiento...}]}]──
```

**Trade-offs**:
- ✅ Simple de implementar — es solo un endpoint REST más
- ✅ Sin estado en el servidor — stateless, fácil de escalar
- ❌ Latencia de hasta 30-60 segundos para recibir una notificación
- ❌ Requests innecesarios cuando no hay notificaciones nuevas

**¿Por qué es suficiente para V1?**: las notificaciones de desplazamiento no son time-critical al nivel de milisegundos. Si Sofía es desplazada a las 14:30, enterarse a las 14:31 es razonablemente instantáneo para un usuario humano.

### Opción B: Server-Sent Events (SSE)

El cliente abre una conexión HTTP persistente. El servidor envía eventos cuando ocurren.

```
Cliente ──[GET /events]──▶ Servidor
         ◀──[data: {type: 'notification', ...}]──  (push inmediato)
```

**Trade-offs**:
- ✅ Notificaciones inmediatas (push real)
- ✅ Más simple que WebSockets (unidireccional, compatible con cualquier proxy HTTP)
- ❌ Mantiene una conexión abierta por usuario — con 25 usuarios, 25 conexiones persistentes (manejable)
- ❌ Más complejidad de implementación que polling

**Recomendación**: implementar SSE en V2 si el polling de 30s resulta demasiado lento en uso real.

### Opción C: WebSockets

Comunicación bidireccional en tiempo real. Necesario para aplicaciones tipo chat o colaboración en tiempo real.

**¿Por qué no aquí?**: las notificaciones solo fluyen del servidor al cliente. La bidireccionalidad de WebSockets no se necesita — SSE es suficiente y más simple.

---

## Diseño de la API REST

### Principios clave

**1. Autenticación: JWT en header Authorization**

```http
POST /reservas
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json

{
  "sala_id": "uuid",
  "fecha_inicio": "2026-06-10T10:00:00Z",
  "fecha_fin": "2026-06-10T11:00:00Z",
  "tipo_reunion": "cliente_externo"
}
```

**2. Respuestas de error consistentes**

```json
// 409 Conflict — reserva imposible
{
  "error": "ROOM_CONFLICT",
  "message": "La sala ya tiene una reserva prioritaria en ese horario",
  "alternatives": [
    {"sala_id": "uuid", "nombre": "Sala Verde", "inicio": "2026-06-10T11:30:00Z"}
  ]
}

// 403 Forbidden — sin permiso
{
  "error": "INSUFFICIENT_PERMISSIONS",
  "message": "Solo el rol gerente puede marcar reservas como urgentes"
}
```

**3. Idempotencia en operaciones críticas**

Las creaciones de reserva deben ser idempotentes ante retries del cliente. Implementar con `Idempotency-Key` en el header o con un `client_request_id` en el body.

---

## Formato de datos: JSON

JSON es la elección correcta aquí. Protocol Buffers añaden complejidad de compilación de schemas sin beneficio para este volumen. Avro está diseñado para streaming de datos (Kafka) — no aplica.

**Formatos de fecha**: ISO 8601 con zona horaria UTC en todos los timestamps (`2026-06-10T10:00:00Z`). La conversión a zona horaria local la hace el frontend. Esto evita bugs de DST (Daylight Saving Time) en el servidor.
