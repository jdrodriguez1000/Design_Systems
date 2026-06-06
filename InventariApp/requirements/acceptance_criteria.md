# Acceptance Criteria — 020 Specification
Fecha: 2026-06-01
Estado: DRAFT
Basado en: specification/spec_analysis_report.md, specification/bdd_features.md, specification/data_contracts.md

---

## Propósito

Este documento define los criterios de aceptación formales por funcionalidad. Cada criterio
es verificable, trazable a un escenario BDD del `bdd_features.md` y a una regla de negocio
del `data_contracts.md`. El criterio de Done del Sprint Contract se verifica al final de
este documento.

---

## AC-F01 — Registro de Entrada de Mercancía

**Fuente BDD:** SC-01
**Actor:** AC-01 — Andrés Mora (Almacenista)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F01-01 | El sistema permite registrar un movimiento de entrada para un producto activo seleccionando: tipo (compra / devolución de cliente), cantidad, fecha/hora, referencia opcional y nota opcional. | Todos los campos obligatorios aceptan valores válidos y el formulario se puede enviar sin error. |
| AC-F01-02 | Al guardar una entrada, el stock actual del producto aumenta exactamente en la cantidad indicada. | El stock actual calculado tras el registro es igual al stock anterior más la cantidad ingresada. |
| AC-F01-03 | El movimiento de entrada queda registrado en el historial de movimientos del producto de forma inmutable. | El movimiento aparece en el historial; no existe ninguna opción de editar ni eliminar ese movimiento. |
| AC-F01-04 | Si el nuevo stock actual supera el stock mínimo y existía una alerta activa para ese producto, la alerta cambia de estado a resuelta automáticamente. | Antes: alerta activa. Después de la entrada: alerta resuelta; no requiere acción manual del usuario. |
| AC-F01-05 | El sistema bloquea el registro de entrada a un producto con estado discontinuado. | Al seleccionar un producto discontinuado, el sistema muestra mensaje explicativo y no crea el movimiento. (RN-03, SE-05) |

---

## AC-F02 — Registro de Salida de Mercancía (camino feliz)

**Fuente BDD:** SC-02
**Actor:** AC-01 — Andrés Mora (Almacenista)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F02-01 | El sistema permite registrar un movimiento de salida para un producto activo cuando la cantidad no supera el stock actual. Campos: tipo (despacho / pérdida-daño), cantidad, fecha/hora, referencia opcional, nota opcional. | Formulario enviable sin error cuando cantidad ≤ stock actual. |
| AC-F02-02 | Al guardar una salida, el stock actual del producto disminuye exactamente en la cantidad indicada. | Stock calculado tras registro = stock anterior − cantidad. |
| AC-F02-03 | El movimiento de salida queda en el historial de forma inmutable. | Movimiento visible en historial; sin opción de editar ni eliminar. |
| AC-F02-04 | Si el nuevo stock actual llega o cae por debajo del stock mínimo (y el stock mínimo es > 0), el sistema genera automáticamente una alerta de stock bajo. | Alerta activa aparece en la pantalla principal sin acción manual. (RN-14, SE-04) |
| AC-F02-05 | Si el stock actual llega exactamente a 0 y el stock mínimo es > 0, se genera alerta de stock bajo. | Verificado en SE-04: salida que deja stock = 0 con mínimo > 0 dispara alerta. |
| AC-F02-06 | Si el stock mínimo del producto es 0, no se genera alerta de stock bajo aunque el stock actual llegue a 0. | Ninguna alerta activa para ese producto después de la salida. (RN-02, RN-16) |

---

## AC-F03 — Contra-Movimiento (corrección de errores)

**Fuente BDD:** SC-03
**Actor:** AC-01 — Andrés Mora (Almacenista)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F03-01 | El sistema permite registrar un contra-movimiento opuesto a un movimiento previo, con referencia al movimiento de origen y nota explicativa obligatoria. | El contra-movimiento se registra; el campo nota es obligatorio y el formulario no se puede enviar sin él. |
| AC-F03-02 | El historial del producto muestra el movimiento original y el contra-movimiento como registros vinculados y visibles. | Ambos movimientos aparecen en el historial; el movimiento original no desaparece ni es modificado. |
| AC-F03-03 | El stock actual queda corregido con el efecto neto de ambos movimientos. | Stock calculado = suma de todos los movimientos incluyendo el contra-movimiento. |
| AC-F03-04 | Ningún movimiento previo puede ser eliminado ni editado directamente. | No existe en la interfaz ningún botón o acción de editar o eliminar movimientos. (RN-07) |

---

## AC-F04 — Alertas de Stock Bajo en Pantalla Principal

**Fuente BDD:** SC-04
**Actor:** AC-01 — Andrés Mora (Almacenista)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F04-01 | Al acceder a la pantalla principal, se muestran todas las alertas de stock bajo activas en ese momento sin necesidad de navegar a otra sección. | Las alertas activas son visibles en la pantalla principal inmediatamente al cargar la vista. |
| AC-F04-02 | Cuando no existen alertas activas, la sección de alertas muestra un estado vacío o ausente. | No se muestra ninguna alerta ficticia; el estado es consistente con la realidad del inventario. |

---

## AC-F05 — Historial de Movimientos por Producto

**Fuente BDD:** SC-05
**Actor:** AC-01 — Andrés Mora (Almacenista)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F05-01 | Al seleccionar un producto y acceder a su historial, se muestra la lista completa de movimientos en orden cronológico descendente. | Los movimientos aparecen del más reciente al más antiguo. |
| AC-F05-02 | Cada movimiento en el historial muestra: fecha/hora del evento, tipo, cantidad, referencia (si existe) y nota (si existe). | Todos estos campos son visibles en el historial de cada movimiento. |
| AC-F05-03 | Los movimientos registrados como excepción (despacho en rojo) aparecen con un indicador visual que los distingue del resto. | Los movimientos con flag excepción = sí tienen marcación visual diferenciada. (RN-11, SE-11) |

---

## AC-F06 — Inventario en Tiempo Real para Diana

**Fuente BDD:** SC-06
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F06-01 | Diana accede al panel de estado general o lista de productos desde su laptop (dentro o fuera de la oficina) con un navegador web. | La interfaz es accesible desde navegador web sin instalación adicional; funciona de forma responsive. |
| AC-F06-02 | El stock actual mostrado a Diana es el mismo que ve Andrés en tiempo real. | Ambos usuarios ven el mismo valor de stock actual para el mismo producto en el mismo momento; no hay desactualización manual. |

---

## AC-F07 — Vista de Alerta con Contexto de 30 Días

**Fuente BDD:** SC-07
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F07-01 | Al seleccionar una alerta activa, la vista muestra el stock actual del producto, el stock mínimo configurado y los movimientos de los últimos 30 días para ese producto. | Los tres elementos están presentes en la vista de alerta. |
| AC-F07-02 | La vista de alerta no incluye valoración monetaria de ningún tipo. | No aparecen precios, costos ni valores monetarios. (EX-01) |

---

## AC-F08 — Configuración de Stock Mínimo en Autoservicio

**Fuente BDD:** SC-08, SE-07, SE-08
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F08-01 | Diana puede modificar el stock mínimo de cualquier producto activo directamente en la interfaz, sin necesidad de intervención de IT. | El campo stock mínimo es editable en la interfaz y el cambio se guarda al confirmar. |
| AC-F08-02 | Si el nuevo stock mínimo genera o resuelve una alerta activa, el sistema recalcula el estado de alerta automáticamente. | (a) Si nuevo mínimo ≤ stock actual: alerta activa se resuelve. (b) Si nuevo mínimo > stock actual: alerta se genera o persiste. (RN-14, RN-15, SE-07) |
| AC-F08-03 | Al configurar el stock mínimo en 0, el sistema muestra una advertencia en la interfaz indicando que el producto nunca generará alerta de stock bajo. | La advertencia aparece antes de guardar cuando el valor ingresado es 0. (SE-08) |
| AC-F08-04 | Si Diana confirma stock mínimo = 0, el producto no genera alerta de stock bajo bajo ninguna circunstancia. | El producto no genera alerta aunque su stock actual llegue a 0. (RN-02, RN-16) |

---

## AC-F09 — Historial de Excepciones

**Fuente BDD:** SC-09, SE-11
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F09-01 | El sistema provee una vista dedicada "Historial de Excepciones" accesible desde el menú principal de navegación. | La vista existe y es accesible desde el menú sin pasos adicionales. (DD-03) |
| AC-F09-02 | El historial de excepciones muestra todas las salidas registradas como excepción, con: nota de motivo, fecha/hora del evento y producto al que corresponde. | Los tres campos están presentes para cada excepción listada. |
| AC-F09-03 | En el historial de movimientos de un producto específico, las excepciones aparecen con indicador visual diferenciado. | Movimientos con flag excepción = sí tienen marcación visual distinta al resto. (SE-11) |

---

## AC-F10 — Generación de Orden de Reposición desde Alerta

**Fuente BDD:** SC-10, SE-09, SE-10
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F10-01 | Diana puede generar una orden de reposición desde una alerta activa con al menos un proveedor asociado al producto. | El formulario de orden se abre con producto y proveedor pre-llenados. |
| AC-F10-02 | La cantidad sugerida se calcula como (stock objetivo − stock actual) y es ajustable por Diana antes de confirmar. | El campo cantidad sugerida está pre-llenado con el cálculo correcto y es editable. |
| AC-F10-03 | La orden queda creada en estado Borrador. | El estado de la orden recién creada es "Borrador". (RE-05, RN-19) |
| AC-F10-04 | El sistema no permite confirmar una orden con cantidad ajustada igual a 0. | Al intentar confirmar con cantidad = 0, el sistema muestra error de validación y la orden permanece en Borrador. (RN-17, SE-09) |
| AC-F10-05 | Al discontinuar un producto con una orden Confirmada pendiente, el sistema muestra advertencia y permite continuar; la orden permanece en estado Confirmada. La cancelación de órdenes está fuera de scope v1 y no es verificable en esta versión. | La advertencia aparece antes de confirmar la discontinuación. La orden no cambia de estado automáticamente. No existe UI ni criterio verificable para cancelar la orden en v1. (RN-20, SE-10) |

---

## AC-F11 — Ciclo de Vida de la Orden de Reposición

**Fuente BDD:** SC-11, SC-12
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F11-01 | Diana puede confirmar una orden en estado Borrador con cantidad > 0; la orden pasa a estado Confirmada. | El estado cambia de Borrador a Confirmada tras la confirmación exitosa. |
| AC-F11-02 | Diana puede marcar una orden Confirmada como Recibida. Al hacerlo, el sistema crea automáticamente un movimiento de entrada por la cantidad de la orden. | El movimiento de entrada aparece en el historial del producto; la cantidad es exactamente la cantidad de la orden. (RN-18, RE-07) |
| AC-F11-03 | Al marcar la orden como Recibida, el sistema reevalúa si la alerta de stock bajo del producto debe resolverse. | Si el nuevo stock supera el mínimo, la alerta se resuelve automáticamente. |
| AC-F11-04 | Las transiciones de estado de la orden son únicamente: Borrador → Confirmada → Recibida. No son reversibles. | No existe ninguna opción de regresar una orden de Confirmada a Borrador ni de Recibida a Confirmada. |

---

## AC-F12 — KPIs Ejecutivos en el Panel de Estado General

**Fuente BDD:** SC-13, SE-12, SE-13
**Actor:** AC-03 — Luis Pedraza (Gerente General)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F12-01 | Al acceder al panel de estado general, se muestra de forma prominente el KPI de productos con alerta de stock bajo activa en ese momento. | El conteo de alertas activas es visible y prominente al cargar el panel. |
| AC-F12-02 | Al acceder al panel de estado general, se muestra el KPI de despachos registrados como excepción en el mes en curso. | El conteo de excepciones del mes actual es visible en el panel. |
| AC-F12-03 | Los KPIs son conteos numéricos sin valoración monetaria. | No aparecen precios, costos ni valores monetarios. (EX-01) |
| AC-F12-04 | Cuando el sistema detecta la condición de "datos no conciliados", se muestra un banner prominente ANTES de los KPIs con el texto exacto: "Advertencia: el inventario puede contener datos no conciliados. Verifique el historial antes de tomar decisiones." | El banner aparece antes que los KPIs; Luis no ve los KPIs sin haber visto primero la advertencia. (DD-02, SE-12) |
| AC-F12-05 | La condición de "datos no conciliados" se activa cuando: (a) existen movimientos con fecha retroactiva (diferencia > 1 hora entre fecha del evento y timestamp de creación) en las últimas 24 horas, O (b) al menos un producto activo tiene stock actual negativo. | Ambas condiciones, por separado, activan el banner. |

---

## AC-F13 — Filtrado de Productos en el Panel de Estado General

**Fuente BDD:** SC-14, SE-13
**Actor:** AC-03 — Luis Pedraza (Gerente General)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F13-01 | La tabla de productos del panel de estado general es filtrable por categoría. | Al seleccionar una categoría, solo aparecen los productos de esa categoría. |
| AC-F13-02 | La tabla es filtrable por estado del producto: OK, ALERTA, SIN MOVIMIENTOS. | Al seleccionar un estado, solo aparecen los productos con ese estado. |
| AC-F13-03 | Un producto activo sin ningún movimiento registrado aparece con estado SIN MOVIMIENTOS y stock actual = 0. | El producto no aparece con estado ALERTA aunque su stock mínimo sea > 0. (SE-13, EE-07) |

---

## AC-F14 — Validaciones de Fecha/Hora en Movimientos

**Fuente BDD:** SE-03, SE-06
**Actor:** AC-01 — Andrés Mora (Almacenista)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F14-01 | El sistema bloquea el registro de cualquier movimiento con fecha/hora futura (posterior a la fecha/hora actual del servidor al guardar). | Al ingresar fecha/hora futura, el sistema muestra error de validación y no crea el movimiento. (RN-08, SE-06) |
| AC-F14-02 | El sistema permite registrar movimientos con fecha/hora retroactiva sin límite mínimo de antigüedad. | Movimientos con fecha retroactiva de cualquier fecha pasada se registran sin error. |
| AC-F14-03 | Cuando la diferencia entre la fecha/hora del evento y el timestamp de creación es mayor a 1 hora, el sistema activa automáticamente el flag de auditoría retroactiva. | El movimiento queda marcado con flag auditoría retroactiva = sí. (RN-09, SE-03) |
| AC-F14-04 | El timestamp de creación es generado automáticamente por el sistema y no es editable por el usuario. | No existe campo de timestamp de creación editable en el formulario de movimientos. |

---

## AC-F15 — Override (Salida con Stock Insuficiente)

**Fuente BDD:** SE-01, SE-02
**Actor:** AC-01 — Andrés Mora (Almacenista)

| ID      | Criterio de aceptación | Condición de verificación |
|---------|------------------------|--------------------------|
| AC-F15-01 | Cuando la cantidad a despachar supera el stock actual, el sistema bloquea la acción y muestra el stock actual disponible. | El mensaje de error incluye el valor del stock actual disponible. (SE-01) |
| AC-F15-02 | El sistema ofrece la opción de cancelar la operación o forzar la salida mediante override. | Dos opciones visibles: cancelar y forzar (override). |
| AC-F15-03 | El override requiere nota obligatoria de motivo antes de confirmar. | El formulario de override no puede enviarse sin nota. (RN-10) |
| AC-F15-04 | Al confirmar el override, la salida se registra como excepción (flag excepción = sí) en el historial de movimientos y en el historial de excepciones consultable por Diana. | El movimiento tiene flag excepción = sí; aparece en el historial de excepciones de Diana. (RN-11) |
| AC-F15-05 | El stock actual puede quedar en negativo tras un override. | El sistema no bloquea ni revierte un stock resultante negativo. |

---

## Verificación del Criterio de Done del Sprint Contract

| Criterio | Estado | Evidencia |
|----------|--------|-----------|
| Todos los actores del 010 tienen ≥1 escenario BDD de camino feliz | CUMPLIDO | AC-01: SC-01 a SC-05 (5 escenarios). AC-02: SC-06 a SC-12 (7 escenarios). AC-03: SC-13 a SC-14 (2 escenarios). AC-04: excluido por ausencia de interfaz en v1 (declarado explícitamente en bdd_features.md). |
| Todos los ítems PENDIENTE del failure_behavior.md tienen política definida | CUMPLIDO | No había ítems [PENDIENTE] en failure_behavior.md. Los 13 ítems (F-01 a F-05, B-01 a B-08) tienen política definida en error_exception_policy.md (EE-01 a EE-13). Las DDs (DD-01, DD-02, DD-03) fueron resueltas por el writer en bdd_features.md, data_contracts.md y error_exception_policy.md. |
| Sin contradicciones entre artefactos | CUMPLIDO | Cada criterio de aceptación referencia el escenario BDD y la regla de negocio del data_contracts.md correspondiente. Las DDs resueltas (DD-01, DD-02, DD-03) son consistentes entre los 4 artefactos. |
| Aprobación explícita del cliente en CP-04 | PENDIENTE | Requiere gate CP-04. |
