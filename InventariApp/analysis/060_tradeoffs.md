# Decisiones y Trade-offs — InventariApp

*Como se aprendió en Contador_Calorias: documentar condiciones de revisión en cada decisión de diseño. Una decisión sin condición de revisión envejece sin actualizarse.*

---

## Decisión 1: Stock calculado vs. stock almacenado

**El problema**: ¿el stock actual de un producto se calcula en tiempo real desde la suma de movimientos, o se mantiene como un campo separado actualizado transaccionalmente?

**Opción A: Stock calculado (recomendado)**
- ✅ Consistencia garantizada por diseño: imposible que el saldo y el historial queden desincronizados
- ✅ El historial es la única fuente de verdad — sin redundancia de datos
- ✅ Simplifica correcciones: un contra-movimiento afecta automáticamente el stock visible
- ❌ La query de panel (stock de todos los productos) es un JOIN + GROUP BY sobre la tabla de movimientos — potencialmente lento con millones de movimientos

**Opción B: Stock almacenado (campo `stock_actual` en productos)**
- ✅ La lectura del stock es O(1) — un simple SELECT del campo
- ✅ El panel de productos es extremadamente rápido sin importar cuántos movimientos haya
- ❌ Requiere actualizar el campo en la misma transacción que el movimiento — si falla, desincronización
- ❌ Dos fuentes de verdad: el campo y el historial pueden divergir con bugs

**Decisión tomada**: Opción A — stock calculado.

**Criterio desempate**: RN-01 lo requiere explícitamente. Además, la escala del sistema (máximo ~100,000 movimientos en 5 años para 500 productos) hace que la query calculada responda en <10ms con índices correctos.

**Condición para reconsiderar**: si la query de panel supera 500ms en P95 con datos reales, evaluar la opción B con actualización transaccional del campo (no como campo editable por la API — solo actualizable internamente en la misma transacción que el movimiento).

---

## Decisión 2: Ausencia de autenticación en v1

**El problema**: el sistema gestiona datos sensibles del negocio (inventario, órdenes, excepciones) pero Luis aceptó operar sin autenticación diferenciada en v1.

**Opción A: Sin autenticación (aceptado para v1)**
- ✅ Desarrollo más rápido — sin el overhead de gestión de usuarios, tokens, refresh
- ✅ Cero fricción para usuarios — sin login, sin contraseñas olvidadas
- ❌ Cualquier usuario con acceso a la URL puede hacer cualquier operación
- ❌ Sin trazabilidad de quién hizo qué (solo qué y cuándo, por el ledger)

**Opción B: JWT con RBAC desde v1**
- ✅ Trazabilidad completa de acciones por usuario
- ✅ Protección contra errores de usuarios no autorizados
- ❌ Añade 2-3 semanas de desarrollo al MVP
- ❌ Introduce complejidad operativa (gestión de contraseñas, tokens, lockouts)

**Decisión tomada**: Opción A — sin autenticación en v1, con restricción de red.

**Criterio desempate**: Luis aceptó el riesgo explícitamente. La mitigación es no exponer el sistema a internet (solo en red interna / VPN). El ledger inmutable garantiza que los errores sean rastreables y corregibles aunque no se sepa quién los cometió.

**Condición para reconsiderar**: antes de que el sistema tenga más de 3 usuarios activos simultáneos, o si alguna acción genera disputas entre usuarios que no se pueden resolver sin saber quién hizo qué.

---

## Decisión 3: Política de override — bloqueo con opción de forzar vs. bloqueo absoluto

**El problema**: cuando el stock es insuficiente para un despacho, ¿el sistema bloquea absolutamente o permite forzar con nota?

**Opción A: Bloqueo con override autorizado (recomendado)**
- ✅ Refleja la realidad del negocio: 2-3 veces/mes hay "mercancía en camino" que justifica despachar sobre stock
- ✅ La excepción queda en el historial con nota de motivo — trazabilidad completa
- ✅ Diana puede revisar el historial de excepciones para detectar patrones de reorden mal calibrado
- ❌ Puede generar stock negativo — estado "innatural" en el sistema
- ❌ Sin auth, cualquier usuario puede forzar el override sin que Diana sea notificada automáticamente

**Opción B: Bloqueo absoluto**
- ✅ Garantiza que el stock nunca quede negativo
- ✅ Fuerza al negocio a tener stock antes de despachar
- ❌ Bloquea operaciones legítimas cuando se sabe que la mercancía llega ese día
- ❌ Genera trabajo manual extra en casos reales del negocio

**Decisión tomada**: Opción A — bloqueo por defecto con override autorizado.

**Criterio desempate**: Diana y Andrés confirmaron explícitamente este escenario (~2-3 veces/mes). Bloquear absolutamente generaría pérdida de ventas reales.

**Condición para reconsiderar**: si los overrides aumentan a >10/mes, indica que los stock mínimos están mal calibrados — en ese caso, el historial de excepciones ya tiene los datos para ajustarlos.

---

## Decisión 4: Alerta como entidad persistente vs. calculada en tiempo real

**El problema**: ¿las alertas son entidades en la base de datos con su propio ciclo de vida, o se calculan en cada request a partir del stock actual y el stock mínimo?

**Opción A: Alerta como entidad persistente (recomendado)**
- ✅ Tiene campos propios: `fecha_disparo`, `stock_al_disparo`, `fecha_resolucion` — información que solo existe en el momento del evento
- ✅ La orden de reposición puede referenciar la alerta de origen (RE-05)
- ✅ La query de alertas activas es O(log n) con índice parcial — muy rápida
- ❌ La alerta puede quedar "huérfana" si hay un bug en la evaluación (ej: stock subió pero la alerta no se resolvió)

**Opción B: Alerta calculada en tiempo real**
- ✅ Siempre consistente — no puede haber alertas huérfanas
- ✅ Sin lógica de ciclo de vida de alertas que mantener
- ❌ La query de "alertas activas" requiere calcular el stock de todos los productos en cada request — potencialmente costosa
- ❌ No hay registro de cuándo se generó la alerta ni el stock en ese momento

**Decisión tomada**: Opción A — alerta como entidad persistente.

**Criterio desempate**: los campos `fecha_disparo` y `stock_al_disparo` son datos de contexto que Diana necesita para la vista de detalle de la alerta (IS-05). Sin entidad persistente, esta información se pierde.

**Condición para reconsiderar**: si se encuentran alertas huérfanas en producción (alertas activas cuyo stock calculado ya supera el mínimo), añadir un job de reconciliación que las resuelva.

---

## Decisión 5: Condición de disparo de "datos no conciliados"

**El problema**: ¿cuándo debe el sistema mostrar el banner de advertencia al Gerente General?

**Opción A: Disparo por movimientos retroactivos en las últimas 24h + stock negativo (recomendado)**
- ✅ Basado en condiciones técnicas objetivas
- ✅ Cubre los dos escenarios más comunes de datos no confiables
- ❌ Puede generar falsos positivos: un movimiento retroactivo legítimo (conciliación normal) activa el banner aunque todo esté bien

**Opción B: Disparo solo cuando stock negativo**
- ✅ Solo activa el banner en condiciones claramente anómalas
- ✅ Menos falsos positivos
- ❌ No cubre el caso de conciliación post-caída donde el stock puede ser positivo pero el historial aún se está completando

**Decisión tomada**: Opción A — ambas condiciones activan el banner (DD-02 resuelto en la especificación).

**Criterio desempate**: Luis prefiere explícitamente ver un aviso cuando hay datos potencialmente no confiables, aunque sea un falso positivo. "Ver un número incorrecto con apariencia de certeza" es peor que ver un aviso preventivo.

**Condición para reconsiderar**: si el banner se activa constantemente en uso normal sin que haya inconsistencias reales, ajustar el umbral de tiempo de movimientos retroactivos (de 1h a 4h, o de 24h de ventana a 6h).

---

## Decisión 6: Timestamp de creación generado en servidor vs. en cliente

**El problema**: el `timestamp_creacion` de cada movimiento, que determina la detección de retroactividad, ¿lo genera el servidor o el cliente?

**Opción A: Generado por el servidor (recomendado)**
- ✅ El cliente no puede falsificar la fecha de creación real
- ✅ El flag de auditoría retroactiva es confiable — no puede desactivarse enviando un timestamp "correcto" desde el cliente
- ❌ El cliente no conoce el timestamp exacto hasta después del INSERT (raro que sea necesario)

**Opción B: Generado por el cliente**
- ✅ El cliente puede incluir el timestamp en la lógica antes del INSERT
- ❌ Un cliente malintencionado puede enviar `timestamp_creacion = fecha_hora_evento` para evitar que el flag retroactivo se active

**Decisión tomada**: Opción A — `DEFAULT NOW()` generado por PostgreSQL.

**Criterio desempate**: la auditoría retroactiva es un mecanismo de integridad, no decorativo. Si el cliente puede falsificarlo, no sirve para nada.

---

## Lecciones transferibles

1. **El stock como valor derivado del historial de movimientos es el patrón correcto para sistemas de inventario**. No almacenar el saldo directamente elimina toda una clase de inconsistencias.

2. **Las operaciones de excepción (override) son parte del diseño, no casos de error**. Sistemas de negocio reales tienen casos que "no deberían pasar" pero pasan 2-3 veces por mes. Diseñar el camino de excepción con trazabilidad es mejor que bloquearlo absolutamente.

3. **La ausencia de autenticación es una decisión de producto, no de ingeniería**. El ingeniero puede documentar el riesgo, pero si el stakeholder lo acepta, el sistema puede operar correctamente con esa restricción — con mitigaciones apropiadas (restricción de red).

4. **Las condiciones de advertencia para el usuario ejecutivo son más valiosas que datos con apariencia de certeza incorrecta**. Luis prefiere un aviso preventivo a un número incorrecto presentado con confianza.
