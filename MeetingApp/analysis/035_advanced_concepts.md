# Conceptos Avanzados — MeetingApp

## La diferencia clave con Contador_Calorias: aquí la consistencia sí importa críticamente

*En Contador_Calorias, se eligió AP (disponibilidad + consistencia eventual) porque el dominio era tolerante a inconsistencias temporales. Aquí el análisis es diferente: un sistema de reservas de salas no puede tener inconsistencias, ni temporales.*

---

## Teorema CAP: CP (Consistencia + Tolerancia a Particiones)

**¿Qué significa CP para MeetingApp?**

El sistema prioriza que todos los nodos vean los mismos datos (consistencia fuerte) sobre la disponibilidad absoluta. Si hay una partición de red entre el servidor de aplicación y la base de datos, el sistema deja de responder (no disponible) en lugar de permitir que operaciones que podrían generar inconsistencias continúen.

**¿Por qué CP y no AP aquí?**

El costo de una inconsistencia en este dominio es concreto:
- Dos personas aparecen en la misma sala al mismo tiempo: conflicto social real
- Una reserva prioritaria no desplazó a la estándar: reunión con cliente externo sin sala
- El desempate de prioridad máxima eligió la reserva equivocada: injusticia percibida

Ninguno de estos es aceptable, incluso temporalmente. La regla de oro: **si el costo de la inconsistencia temporal es alto y visible, elegir CP**.

Ejemplos reales CP: sistemas bancarios, inventario de e-commerce, tickets de conciertos (no puedes vender el mismo asiento dos veces).

---

## Modelo de consistencia: Consistencia Fuerte (por transacción)

Cada operación de escritura (crear reserva, ejecutar desplazamiento) ocurre dentro de una transacción ACID que garantiza que:
1. Nadie ve el estado intermedio
2. O toda la operación tiene efecto, o ninguna parte tiene efecto
3. El estado tras la transacción es consistente con todas las reglas de negocio

**Implementación concreta**:

```sql
BEGIN;
-- Lock de la sala: ninguna otra transacción puede modificarla hasta el COMMIT
SELECT id FROM salas WHERE id = $sala_id FOR UPDATE;

-- Verificar solapamiento (lee estado consistente — nadie puede insertar en paralelo)
SELECT id, tipo_prioridad, timestamp_creacion
FROM reservas
WHERE sala_id = $sala_id
  AND estado = 'activa'
  AND fecha_inicio < $fin
  AND fecha_fin > $inicio;

-- Según resultado: insertar, desplazar, o rechazar
-- ...todo en la misma transacción

COMMIT;
-- Aquí el lock se libera y otras transacciones pueden continuar
```

---

## Alta Disponibilidad

**SLA esperado**: para una empresa de 25 personas, la disponibilidad objetivo es ~99% (máximo ~7 horas de downtime por mes). Esto es razonablemente alcanzable con:
- Un servidor con reinicio automático (PM2, Docker restart policy)
- Backups diarios de la BD
- Procedimiento de fallback documentado (el WhatsApp con Ana — ya está en el diseño como SF-03)

**El fallback ya está diseñado**: la administradora actúa como punto de coordinación manual durante caídas. Esto es una decisión de arquitectura de negocio que reduce la presión sobre la disponibilidad técnica. No se necesita un 99.9% de uptime si hay un proceso de contingencia claro.

---

## Tolerancia a fallos

### Circuit Breaker — No aplica en V1

El Circuit Breaker protege a un servicio de llamar repetidamente a un servicio caído. Con un monolito y una sola BD, no hay servicios externos que proteger. Si se añade en el futuro un servicio de correo (DF-02) o calendarios (DF-03), el Circuit Breaker protegería al monolito de que una falla en el servicio externo bloquee las reservas.

### Retry Policy — Aplica para escrituras en BD

Las transacciones de PostgreSQL pueden fallar por deadlocks (cuando dos transacciones intentan lockear las mismas filas en orden opuesto). La política correcta:
- Reintentar automáticamente hasta 3 veces con backoff exponencial (100ms, 200ms, 400ms)
- Si falla tras 3 intentos, devolver error al usuario

```python
def execute_with_retry(operation, max_retries=3):
    for attempt in range(max_retries):
        try:
            return operation()
        except DeadlockDetected:
            if attempt == max_retries - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))  # 100ms, 200ms, 400ms
```

### Bulkhead Pattern — Parcialmente aplicable

El Bulkhead aísla recursos para que el fallo de un componente no afecte a otros. En este monolito: usar un pool de conexiones a DB limitado para evitar que un burst de requests agote las conexiones y deje al sistema sin poder responder a nada.

---

## ACID vs BASE

### ACID — Aplica a todas las operaciones de reserva

**Atomicidad**: si el desplazamiento falla a mitad (reserva cancelada pero nueva no creada), la transacción revierte. No queda la sala en estado fantasma.

**Consistencia**: las restricciones de la BD (CHECK constraints, FOREIGN KEY) garantizan que el estado siempre cumple las reglas de negocio, incluso si hay un bug en la aplicación.

**Aislamiento**: `SELECT FOR UPDATE` garantiza que dos transacciones concurrentes no ven el mismo estado "libre" al mismo tiempo.

**Durabilidad**: PostgreSQL escribe a disco (WAL) antes de confirmar el COMMIT. Si el servidor cae después del COMMIT, los datos están guardados.

### BASE — No aplica aquí

El modelo BASE (Básicamente disponible, Estado suave, Consistencia eventual) se usa en sistemas donde la consistencia inmediata no es crítica (carritos de compra, feeds de redes sociales, contadores de likes). Para un sistema de reservas donde dos personas no pueden estar en la misma sala, BASE es inapropiado.
