# Estrategias de Escalabilidad — InventariApp

## Contexto: la escala más pequeña que hemos diseñado hasta ahora

18 empleados, 1 almacén, 1 operador principal (Andrés). Los números concretos:

- **Usuarios concurrentes máximos**: 4-5 (Andrés + Diana + Luis + ocasionalmente 1-2 más)
- **Movimientos por día**: ~20-50 (en empresa de 18 personas con distribución de insumos de oficina)
- **Requests por día**: ~200-400 (movimientos + consultas del panel + alertas)
- **Tamaño de la BD en 5 años**: ~100,000 movimientos, ~500 productos, ~200 órdenes/año → total ~50,000 filas adicionales/año

A esta escala, **la escalabilidad no es un reto técnico** y no debería serlo. Un servidor con 1 CPU y 512MB de RAM maneja esto con 90% de recursos ociosos. Esta sección es educativa: documenta qué cambiaría si el sistema creciera.

---

## Escalabilidad actual: vertical mínima

**1 servidor + PostgreSQL** es todo lo necesario.

¿Por qué no horizontal desde el inicio? El horizontal scaling introduce complejidad de estado compartido entre instancias. Para InventariApp, el estado más sensible es la evaluación de alertas: si dos instancias del backend procesan movimientos simultáneamente para el mismo producto, podrían intentar crear dos alertas activas simultáneas (violando RN-13).

Con una sola instancia, este problema no existe. El costo de agregar complejidad de coordinación para proteger una empresa de 18 usuarios es injustificado.

*Como se aprendió en MeetingApp: el horizontal scaling introduce el problema del estado compartido en los locks de base de datos. Para esta escala, añadir esta complejidad es sobrediseño.*

---

## El único cuello de botella real: cálculo de stock

La query que más escala necesitará eventualmente es el cálculo de stock actual de todos los productos para el panel de estado:

```sql
SELECT
    p.id,
    p.nombre,
    COALESCE(SUM(
        CASE WHEN m.tipo_impacto = 'SUMA' THEN m.cantidad
             ELSE -m.cantidad END
    ), 0) AS stock_actual
FROM productos p
LEFT JOIN movimientos m ON m.producto_id = p.id
GROUP BY p.id, p.nombre;
```

Con 500 productos y 100,000 movimientos, esta query ejecuta en <10ms con el índice correcto en `movimientos(producto_id)`. Sin índice, haría un seq scan de toda la tabla de movimientos — todavía aceptable para este tamaño, pero mala práctica.

**Si el negocio crece (200 empleados, 5,000 movimientos/día):**

Dos estrategias complementarias:

1. **Snapshot de saldo por producto**: mantener un campo `stock_snapshot` actualizado transaccionalmente en cada movimiento. En lugar de agregar todo el historial, se suma el snapshot + movimientos recientes.

2. **Vista materializada en PostgreSQL**: `REFRESH MATERIALIZED VIEW CONCURRENTLY stocks_actuales` ejecutado en background cada N segundos. El panel lee de la vista; los movimientos individuales siguen calculando desde el historial.

---

## Estrategia de caché

**No necesaria en v1.**

**Si se añade en v2:**
- Redis con TTL de 30-60 segundos para los KPIs del panel ejecutivo (alertas activas + excepciones del mes)
- Invalidación: al insertar cualquier movimiento o cambiar el estado de una alerta
- Para 3-5 consultas del panel por hora, el beneficio de Redis no justifica su operación

---

## Load Balancing

**No aplica en v1.**

Si se despliegan múltiples instancias en el futuro (ej: alta disponibilidad con failover), el algoritmo es round-robin con sticky sessions para el frontend. La restricción es que el locking de PostgreSQL (`SELECT FOR UPDATE` en la evaluación de alertas) funciona correctamente con múltiples instancias apuntando a la misma BD.

---

## Database Scaling

**Replicación read/write:** Si las consultas del panel se vuelven costosas (empresa con miles de productos y millones de movimientos), añadir una réplica de lectura. El panel ejecutivo y el historial de movimientos van a la réplica. Los inserts de movimientos van al primary.

**¿Sharding?**: No aplicable. El sharding tiene sentido cuando una sola BD no puede manejar el volumen de escrituras. InventariApp estaría a órdenes de magnitud de ese punto incluso con 10x el tamaño esperado.

---

## Estimaciones de capacidad

| Escenario | Movimientos/día | Tamaño BD (5 años) | Servidor necesario |
|-----------|----------------|---------------------|-------------------|
| **Actual** (18 usuarios) | ~30 | ~55,000 filas | 1 CPU, 512MB RAM |
| **Crecimiento 5x** (90 usuarios) | ~150 | ~275,000 filas | 1 CPU, 1GB RAM |
| **Crecimiento 20x** (360 usuarios) | ~600 | ~1,100,000 filas | 2 CPUs, 2GB RAM + cache |

La conclusión es que este sistema puede manejar un crecimiento significativo sin cambios arquitectónicos. Solo cuando supere los 100,000 movimientos/día sería necesario replantear el diseño del cálculo de stock.

---

## Lección de escala

**La regla de los 3 órdenes de magnitud**: diseña para 100x tu escala actual con los mismos patrones, y solo replantea cuando llegues al límite real. Con 18 usuarios, diseña con la cabeza puesta en 1,800. A 1,800, entonces diseñas para 180,000. El uso real revela los cuellos de botella reales — mucho mejor que la especulación.

Sistemas reales que siguieron este camino: Shopify empezó como un monolito Rails con SQLite y llegó a manejar millones de tiendas antes de extraer servicios específicos. El monolito no fue el problema — fue la base que permitió crecer.
