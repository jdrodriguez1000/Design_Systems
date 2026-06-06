# Estrategias de Escalabilidad — MeetingApp

## Contexto: escala pequeña y bien acotada

25 empleados, 3 salas, ~10-50 reservas por día. Los números concretos:
- **Usuarios concurrentes máximos**: ~10 (reunión de empresa completa)
- **Reservas activas simultáneas**: máximo 3 (una por sala)
- **Requests por día**: ~200-500 (consultas + reservas + notificaciones)
- **Tamaño de la BD en 5 años**: ~100,000 filas (incluyendo historial de notificaciones)

A esta escala, **la escalabilidad no es un reto técnico**. Un servidor con 1 CPU y 512MB de RAM maneja esto con margen amplio. Esta sección es educativa: documenta qué cambiaría si el sistema creciera.

---

## Escalabilidad actual: vertical simple

**1 servidor + PostgreSQL** es todo lo necesario. No hay load balancer, no hay réplicas de lectura, no hay caché.

**¿Por qué no horizontal scaling desde el inicio?**

El horizontal scaling introduce estado compartido entre instancias. El estado más crítico aquí es el locking de base de datos (`SELECT FOR UPDATE`). Con una sola instancia, el lock es local. Con múltiples instancias, el lock debe ser a nivel de BD — que PostgreSQL ya maneja, pero añade coordinación. Para 25 usuarios, añadir esta complejidad sin necesidad es sobrediseño.

---

## Cuello de botella anticipado bajo carga

Si la empresa crece a 200 empleados con 20 salas, el cuello de botella sería la **query de detección de solapamiento**:

```sql
SELECT COUNT(*) FROM reservas
WHERE sala_id = $1
  AND estado = 'activa'
  AND NOT (fecha_fin <= $2 OR fecha_inicio >= $3);
```

Esta query se ejecuta con `SELECT FOR UPDATE` — bloquea la fila de la sala durante la transacción. Con muchos usuarios concurrentes intentando reservar la misma sala, las transacciones se quedan en cola esperando el lock. La solución: **particionamiento de la tabla de reservas por sala** y **réplicas de lectura** para las consultas de disponibilidad.

---

## Estrategia de caché — no necesaria en V1

Las consultas de disponibilidad son relativamente infrecuentes (25 usuarios, máximo ~50 consultas/hora). El costo de invalidar un caché cuando una reserva cambia supera el beneficio.

**Si se añadiera caché**: Redis con TTL corto (5-10 segundos) para el estado de disponibilidad de las salas. La invalidación ocurre al crear/cancelar cualquier reserva. Para 3 salas, son solo 3 entradas en Redis.

---

## Load Balancing — fase futura

Si se despliegan múltiples instancias del backend (ej: si el sistema se expande a múltiples oficinas), el algoritmo recomendado es **round-robin con sticky sessions** para los WebSockets/SSE (si se implementa en V2).

---

## Database scaling

**Replicación read/write**: si las consultas de disponibilidad se vuelven costosas, añadir una réplica de lectura (PostgreSQL streaming replication). Las escrituras van al primary, las lecturas de disponibilidad van a la réplica.

**¿Sharding?**: no aplicable para este tamaño. El sharding tiene sentido cuando una sola BD no puede manejar el volumen de escrituras — aquí estamos a órdenes de magnitud de ese punto.

---

## Auto-scaling

Para archivos estáticos del frontend (si se usa CDN): auto-scaling automático por definición — CDN maneja el tráfico.

Para el backend: en Railway/Render/Fly.io, el auto-scaling vertical (agrandar el servidor) es manual pero simple. No se necesita Kubernetes para esto.

---

## Lección de escala de sistemas reales

- **Booking.com** procesa ~1.5 millones de reservas de hotel por día con arquitectura similar a este monolito (aunque con mucho más sharding y caché). El punto de partida fue exactamente un monolito Rails.
- **La regla de los 3 órdenes de magnitud**: diseña para 100x tu escala actual. Con 25 usuarios, diseña para 2,500. Con 2,500, diseña para 250,000. Cada vez que llegues al límite, ya tienes la información de uso real para rediseñar correctamente.
