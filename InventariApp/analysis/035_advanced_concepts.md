# Conceptos Avanzados — InventariApp

## CAP Theorem: CP (Consistencia + Tolerancia a Particiones)

**¿Qué significa CP para InventariApp?**

El sistema prioriza que todos los usuarios vean el mismo stock consistente sobre la disponibilidad absoluta. Si hay una partición de red entre el servidor de aplicación y la base de datos, el sistema deja de responder antes de mostrar datos potencialmente incorrectos.

**¿Por qué CP y no AP?**

*Como se aprendió en MeetingApp: elegir AP cuando el dominio requiere CP resulta en bugs que no aparecen en pruebas pero sí en producción. La regla: si el costo de la inconsistencia temporal es alto y visible, elegir CP.*

Para InventariApp el costo de la inconsistencia es concreto:
- Diana ve stock = 10 cuando en realidad hay 0 → hace una promesa de entrega que no puede cumplirse
- Luis ve 0 alertas activas cuando en realidad hay 3 → reporta a socios un estado incorrecto
- El sistema muestra "stock suficiente" cuando Andrés ya hizo un override → Diana no sabe que hay una excepción

Nótese la diferencia con un feed de redes sociales (AP apropiado): si ves un "like" de hace 2 segundos tarde, el costo es cero. Si ves un stock incorrecto y vendes lo que no tienes, el costo es real.

**La única diferencia con MeetingApp:** en MeetingApp la inconsistencia producía un conflicto visible e inmediato (dos personas en la misma sala). Aquí la inconsistencia puede ser más silenciosa (stock desactualizado durante segundos), pero igualmente costosa cuando alguien toma una decisión basada en ese dato.

---

## Modelo de consistencia: Consistencia Fuerte por transacción

Cada operación de escritura que modifica el estado del inventario ocurre dentro de una transacción ACID que garantiza:

1. Nadie ve el movimiento sin que el stock esté actualizado
2. Nadie ve la alerta sin que el movimiento que la causó esté en el historial
3. O toda la operación tiene efecto, o ninguna parte tiene efecto

**El flujo transaccional crítico:**

```sql
BEGIN;
-- 1. Bloquear el producto para evitar evaluaciones concurrentes de alerta
SELECT id, stock_minimo FROM productos WHERE id = $producto_id FOR UPDATE;

-- 2. Insertar el movimiento
INSERT INTO movimientos (...) VALUES (...);

-- 3. Calcular nuevo stock
SELECT COALESCE(SUM(CASE tipo_impacto WHEN 'SUMA' THEN cantidad ELSE -cantidad END), 0)
FROM movimientos WHERE producto_id = $producto_id;

-- 4a. Si nuevo_stock <= stock_minimo Y no hay alerta activa → crear alerta
INSERT INTO alertas_stock_bajo (producto_id, estado, stock_al_disparo, fecha_disparo)
VALUES ($producto_id, 'activa', $nuevo_stock, NOW());
-- 4b. Si nuevo_stock > stock_minimo Y hay alerta activa → resolver alerta
UPDATE alertas_stock_bajo SET estado='resuelta', fecha_resolucion=NOW()
WHERE producto_id = $producto_id AND estado = 'activa';

COMMIT;
```

**¿Por qué `SELECT ... FOR UPDATE` en el producto?**

*Aplicando la lección de MeetingApp: SELECT FOR UPDATE es la herramienta correcta cuando necesitamos leer el estado de una entidad y potencialmente modificarla en la misma transacción, previniendo race conditions.*

En InventariApp, aunque Andrés es el único operador de movimientos, Diana puede marcar una orden como Recibida (lo que genera un movimiento de entrada automáticamente). Si Andrés y Diana ejecutan movimientos sobre el mismo producto simultáneamente:

- Sin `FOR UPDATE`: ambas transacciones podrían intentar crear una alerta activa para el mismo producto, violando el índice único parcial y causando un error
- Con `FOR UPDATE`: la segunda transacción espera hasta que la primera completa, ve el estado actualizado, y toma la decisión correcta

El lock dura milisegundos — imperceptible para el usuario.

---

## ACID — todas las operaciones de inventario

**Atomicidad**: si la inserción del movimiento tiene éxito pero la evaluación de alerta falla, la transacción completa revierte. No queda un movimiento sin alerta cuando debería haberla.

**Consistencia**: los `CHECK` constraints de la BD (cantidad > 0, tipos válidos, nota obligatoria para excepciones) garantizan que el estado siempre cumple las reglas de negocio incluso si hay un bug en la aplicación.

**Aislamiento**: `SELECT FOR UPDATE` en el producto garantiza que dos movimientos concurrentes sobre el mismo producto no evalúan alertas con datos obsoletos.

**Durabilidad**: PostgreSQL escribe al WAL antes de confirmar el COMMIT. Si el servidor cae después del COMMIT, los movimientos y alertas están guardados.

---

## BASE — no aplica aquí

BASE (Básicamente disponible, Estado suave, Consistencia eventual) se usa en sistemas donde la consistencia inmediata no es crítica: carritos de compra de e-commerce, feeds de redes sociales, contadores de likes. Para un sistema de inventario donde el stock incorrecto genera promesas de venta incumplibles, BASE es inapropiado.

---

## Alta Disponibilidad

**SLA esperado**: para una empresa de 18 personas, el objetivo es ~99% de disponibilidad (~7 horas de downtime/mes tolerable). La empresa ya tiene un fallback documentado: papel + WhatsApp con Ana durante caídas (F-03 en failure_behavior.md).

Este fallback cambia el requisito técnico: el sistema no necesita 99.9% de uptime si hay un proceso de contingencia claro. El objetivo técnico real es: **no perder datos tras una caída** y **permitir conciliación retroactiva**.

El sistema ya está diseñado para esto: fecha/hora del evento editable + flag de auditoría retroactiva + historial inmutable como fuente de verdad.

**Estrategia de alta disponibilidad práctica:**
- Un servidor con Docker restart policy `always` (PM2 o systemd para alternativas)
- Backups diarios automatizados de PostgreSQL (pg_dump a S3 o similar)
- El fallback en papel garantiza que las caídas < 1 día no pierdan operaciones del negocio

---

## Tolerancia a fallos

### Retry con backoff exponencial para deadlocks

Si dos movimientos sobre el mismo producto llegan simultáneamente, ambos intentarán `SELECT ... FOR UPDATE` sobre el mismo producto. PostgreSQL gestiona el orden de los locks y uno de los dos puede encontrar un deadlock. La política correcta: reintentar hasta 3 veces con backoff exponencial (100ms, 200ms, 400ms).

*Como se aprendió en MeetingApp: sin retry, un deadlock esporádico se convierte en un error visible para el usuario de una operación que habría tenido éxito en el segundo intento.*

### Circuit Breaker — no aplica en v1

Con un monolito y una sola BD, no hay servicios externos que proteger. Si en el futuro se añade un servicio de email para notificaciones, el Circuit Breaker protegería al monolito de que una falla en el servicio de email bloquee el registro de movimientos.

### Bulkhead — parcialmente aplicable

Usar un pool de conexiones a BD limitado (10-20 conexiones máximo) para evitar que un burst de requests agote las conexiones de PostgreSQL. FastAPI + asyncpg maneja esto con su pool de conexiones asíncrono.
