# Data Contracts — 020 Specification
Fecha: 2026-06-01
Estado: DRAFT
Basado en: specification/spec_analysis_report.md

---

## Entidades y sus Contratos

### Entidad EN-01 — Producto

**Descripción:** Artículo del catálogo del almacén. Tiene un código interno único e inmutable
asignado al crearlo. Su stock actual se calcula siempre a partir del historial de movimientos
y nunca se edita directamente. Puede estar activo o discontinuado. Puede tener hasta dos
proveedores asociados.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| código interno | texto | alfanumérico | sí | Único en el catálogo. Inmutable: no puede modificarse una vez asignado. Se asigna al crear el producto. | SC-01, SC-02, SC-05 |
| nombre | texto | libre | sí | No puede estar vacío. | SC-01, SC-02 |
| categoría | lista (texto) | valor de lista cerrada | sí | Debe corresponder a una categoría existente. Todo producto pertenece a exactamente una categoría. | SC-14, SE-13 |
| unidad de medida | texto | libre (ej: unidad, caja, resma, cartucho) | sí | No puede estar vacío. | SC-01, SC-02 |
| stock mínimo | número entero | entero ≥ 0 | sí | Valor por defecto: 0. Si se configura en 0, el producto nunca genera alerta de stock bajo. Al modificar a 0, el sistema muestra advertencia UI. | SC-08, SE-07, SE-08 |
| stock actual | número entero (calculado) | entero (puede ser negativo tras excepción/despacho en rojo) | no editable | Solo lectura. Se calcula a partir de la suma de todos los movimientos del historial del producto. No se edita directamente. | SC-01, SC-02, SC-03 |
| estado | lista (texto) | activo / discontinuado | sí | Valor inicial: activo. Un producto discontinuado no puede recibir nuevas entradas. Su historial de movimientos se preserva. | SE-05, SE-10 |
| proveedor 1 | referencia a Proveedor | — | no | Opcional. Si existe, debe referenciar un Proveedor válido. | SC-10 |
| proveedor 2 | referencia a Proveedor | — | no | Opcional. Si existe, debe referenciar un Proveedor válido. Un producto puede tener hasta 2 proveedores. | SC-10 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-01 | El stock actual de un producto se calcula siempre a partir del historial de movimientos; nunca se edita directamente. | alta |
| RN-02 | Un producto con stock mínimo = 0 no genera alerta de stock bajo aunque su stock actual llegue a 0. | alta |
| RN-03 | Un producto con estado discontinuado no puede recibir nuevas entradas. Su historial se preserva. | alta |
| RN-04 | El código interno es único e inmutable: no puede cambiar después de la creación del producto. | alta |
| RN-05 | Un producto pertenece a exactamente una categoría. | media |
| RN-06 | Un producto puede tener cero, uno o hasta dos proveedores registrados. | media |

---

### Entidad EN-02 — Movimiento

**Descripción:** Registro inmutable de una transacción de stock. Una vez creado, no puede
eliminarse ni editarse. Puede ser de tipo entrada (compra a proveedor o devolución de cliente),
salida (despacho o pérdida/daño) o contra-movimiento (corrección de un movimiento erróneo).
Un subconjunto de las salidas es la excepción (despacho en rojo / override), registrada cuando
el stock disponible es insuficiente.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| producto | referencia a Producto | — | sí | Debe referenciar un Producto existente. Si el producto está discontinuado, no se puede registrar una entrada. | SC-01, SC-02, SE-05 |
| tipo | lista (texto) | entrada / salida / contra-movimiento | sí | No puede estar vacío. Determina si el stock aumenta (entrada, contra-movimiento que revierte salida) o disminuye (salida, contra-movimiento que revierte entrada). | SC-01, SC-02, SC-03 |
| subtipo | lista (texto) | compra / devolución de cliente / despacho / pérdida-daño | sí para entrada y salida | Complementa el tipo. Para contra-movimientos no aplica subtipo; se infiere del movimiento que revierte. | SC-01, SC-02 |
| cantidad | número entero | entero > 0 | sí | Debe ser mayor que 0. | SC-01, SC-02, SE-01 |
| fecha/hora del evento | fecha y hora | DD/MM/AAAA HH:MM | sí | No puede ser una fecha/hora futura (posterior a la fecha/hora actual del servidor al guardar). Fecha retroactiva permitida sin límite mínimo. | SC-03, SE-03, SE-06 |
| timestamp de creación | fecha y hora (de sistema) | DD/MM/AAAA HH:MM:SS | sí (automático) | Generado automáticamente por el sistema al momento de guardar el movimiento. No editable. | SE-03 |
| flag auditoría retroactiva | booleano | sí / no | sí (automático) | El sistema asigna "sí" automáticamente cuando la diferencia entre fecha/hora del evento y timestamp de creación es mayor a 1 hora. De lo contrario, asigna "no". | SE-03 |
| referencia | texto | libre | no | Opcional. Campo de referencia externa (ej: número de factura, número de remisión). | SC-01, SC-02 |
| nota | texto | libre | no (excepto en excepciones y contra-movimientos) | Obligatoria cuando el tipo es contra-movimiento o cuando flag excepción = sí. | SC-03, SE-02 |
| flag excepción | booleano | sí / no | sí (automático) | El sistema asigna "sí" cuando la salida fue forzada mediante override sobre stock insuficiente. De lo contrario, "no". | SE-02, SC-09 |
| nota de excepción | texto | libre | sí si flag excepción = sí | Obligatoria cuando el flag excepción es "sí". Captura el motivo del override. | SE-02 |
| movimiento de origen | referencia a Movimiento | — | no | Solo aplica para contra-movimientos. Referencia al movimiento original que se está revirtiendo. | SC-03 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-07 | Todo movimiento es inmutable una vez creado. No puede eliminarse ni editarse. La corrección de errores se realiza únicamente mediante contra-movimientos. | alta |
| RN-08 | La fecha/hora del evento no puede ser futura. El sistema valida contra la fecha/hora actual del servidor al momento de guardar. | alta |
| RN-09 | Cuando la fecha/hora del evento difiere en más de 1 hora del timestamp de creación, el sistema activa el flag de auditoría retroactiva automáticamente. | alta |
| RN-10 | La nota es obligatoria en contra-movimientos y en salidas con flag excepción = sí. | alta |
| RN-11 | Un movimiento con flag excepción = sí queda visible en el historial de excepciones consultable por Diana. | alta |
| RN-12 | Al registrar cualquier movimiento, el sistema recalcula el stock actual del producto afectado y evalúa si debe generar o resolver una alerta de stock bajo. | alta |

---

### Entidad EN-03 — Alerta de Stock Bajo

**Descripción:** Condición activa cuando el stock actual de un producto llega o cae por debajo
de su stock mínimo. Persiste hasta que el stock supera el mínimo, momento en que se resuelve
automáticamente. No puede haber más de una alerta activa por producto a la vez.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| producto | referencia a Producto | — | sí | Debe referenciar un Producto existente. Solo puede existir una alerta activa por producto simultáneamente. | SC-01, SC-02, SC-04, SE-04 |
| estado | lista (texto) | activa / resuelta | sí (automático) | El sistema gestiona el estado automáticamente según el stock actual vs. stock mínimo del producto. | SC-01, SC-04, SE-07 |
| stock actual al disparo | número entero | entero ≤ stock mínimo | sí (automático) | Registrado automáticamente cuando la alerta se genera. | SC-02, SE-04 |
| fecha/hora de disparo | fecha y hora | DD/MM/AAAA HH:MM | sí (automático) | Registrada automáticamente cuando la alerta se genera. | SC-02, SE-04 |
| fecha/hora de resolución | fecha y hora | DD/MM/AAAA HH:MM | no (automático) | Registrada automáticamente cuando la alerta se resuelve. Vacía mientras la alerta está activa. | SC-01, SE-07 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-13 | No puede existir más de una alerta activa por producto simultáneamente. Si el stock baja más mientras la alerta ya está activa, la alerta existente persiste sin crear una segunda. | alta |
| RN-14 | La alerta se genera automáticamente cuando el stock actual llega a ser ≤ stock mínimo, siempre que el stock mínimo sea > 0. | alta |
| RN-15 | La alerta se resuelve automáticamente cuando el stock actual supera el stock mínimo del producto. | alta |
| RN-16 | Un producto con stock mínimo = 0 no genera alerta de stock bajo bajo ninguna circunstancia (ver RN-02). | alta |

---

### Entidad EN-04 — Orden de Reposición

**Descripción:** Solicitud formal de compra generada por Diana desde una alerta activa. Tiene
un ciclo de vida de estados: Borrador, Confirmada y Recibida. Al pasar a estado Recibida,
el sistema crea automáticamente un movimiento de entrada por la cantidad de la orden.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| producto | referencia a Producto | — | sí | Debe referenciar un Producto existente. Pre-llenado al generar desde una alerta activa. | SC-10, SC-11, SC-12 |
| proveedor | referencia a Proveedor | — | sí | Debe referenciar un Proveedor asociado al producto. Pre-llenado al generar desde una alerta activa. | SC-10, SC-11 |
| cantidad sugerida | número entero (calculado) | entero > 0 | sí (calculado) | Calculada como (stock objetivo − stock actual). Solo lectura; sirve como referencia. | SC-10 |
| cantidad ajustada | número entero | entero > 0 | sí | Ajustable por Diana antes de confirmar. Debe ser > 0 para poder confirmar la orden. | SC-10, SE-09 |
| estado | lista (texto) | Borrador / Confirmada / Recibida | sí (automático) | Estado inicial: Borrador. Transiciones permitidas: Borrador → Confirmada → Recibida. No reversible. | SC-10, SC-11, SC-12, SE-09, SE-10 |
| fecha/hora de creación | fecha y hora | DD/MM/AAAA HH:MM | sí (automático) | Generada automáticamente por el sistema al crear la orden. | SC-10 |
| alerta de origen | referencia a Alerta de Stock Bajo | — | sí | Debe referenciar la alerta activa que originó la orden. | SC-10 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-17 | Una orden de reposición no puede confirmarse si la cantidad ajustada es 0. | alta |
| RN-18 | Al pasar a estado Recibida, el sistema crea automáticamente exactamente un movimiento de entrada por la cantidad ajustada. | alta |
| RN-19 | Una alerta activa puede originar como máximo una orden de reposición activa (en estado Borrador o Confirmada). | media |
| RN-20 | Si el producto de una orden Confirmada es discontinuado, la orden permanece en estado Confirmada. La transición de estado Confirmada a Cancelada no está implementada en v1 (fuera de scope). Diana debe gestionar la cancelación fuera del sistema. | media |

---

### Entidad EN-05 — Proveedor

**Descripción:** Entidad que suministra un producto. Cada producto puede tener hasta dos
proveedores registrados. Solo el nombre es obligatorio; los datos de contacto son opcionales.
Los proveedores existen únicamente como referencia en el catálogo de productos y en las
órdenes de reposición.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| nombre | texto | libre | sí | No puede estar vacío. | SC-10, SC-11 |
| teléfono | texto | libre | no | Opcional. Sin validación de formato en v1. | SC-10 |
| email | texto | libre | no | Opcional. Sin validación de formato en v1. | SC-10 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-21 | Solo el nombre del proveedor es obligatorio. Los campos teléfono y email son opcionales. | media |
| RN-22 | Una orden de reposición puede generarse aunque el proveedor no tenga datos de contacto completos. Diana gestiona el contacto fuera del sistema. | media |

---

### Entidad EN-06 — Categoría

**Descripción:** Clasificación agrupadora de productos usada en el panel de estado general
para filtrar la tabla de productos. Todo producto pertenece a exactamente una categoría.

**Definición inline:** Agrupación lógica de productos según tipo o familia, usada para filtrar y organizar la vista de inventario en el panel de estado general. El término no figura en el domain_glossary.md del 010 (artefacto cerrado); se define aquí para uso exclusivo de la especificación v1.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| nombre | texto | libre | sí | No puede estar vacío. Único dentro del catálogo de categorías. | SC-14, SE-13 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-23 | Todo producto pertenece a exactamente una categoría (RE-04). | media |

---

## Relaciones entre Entidades

| ID    | Entidad A | Relación | Entidad B | Cardinalidad | Restricción de negocio |
|-------|-----------|----------|-----------|--------------|------------------------|
| RE-01 | Producto | tiene | Movimiento | 1 a 0..N | Un producto recién creado tiene 0 movimientos; su stock actual es 0 |
| RE-02 | Producto | puede tener | Alerta de Stock Bajo activa | 1 a 0..1 | Un producto no puede tener más de una alerta activa simultáneamente; si el stock ya estaba bajo y cae más, la alerta existente persiste y refleja el stock actualizado |
| RE-03 | Producto | se asocia con | Proveedor | 1 a 0..2 | Un producto puede tener cero, uno o hasta dos proveedores registrados |
| RE-04 | Producto | pertenece a | Categoría | N a 1 | Todo producto tiene exactamente una categoría |
| RE-05 | Alerta de Stock Bajo | puede originar | Orden de Reposición | 1 a 0..1 | Una alerta puede originar como máximo una orden de reposición activa |
| RE-06 | Orden de Reposición | referencia | Proveedor | N a 1 | Toda orden de reposición debe referenciar exactamente un proveedor |
| RE-07 | Orden de Reposición (estado Recibida) | genera automáticamente | Movimiento de tipo entrada | 1 a 1 | Al marcar una orden como Recibida, el sistema crea exactamente un movimiento de entrada por la cantidad de la orden |
