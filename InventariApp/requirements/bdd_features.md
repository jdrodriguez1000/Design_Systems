# BDD Feature Files — 020 Specification
Fecha: 2026-06-01
Estado: DRAFT
Basado en: specification/spec_analysis_report.md

---

## Resolución de Decisiones de Diseño

### DD-01 — Campo fecha/hora en movimientos (rangos, validación, auditoría)

La fecha/hora de un movimiento es editable por el usuario para soportar movimientos retroactivos
(caídas del sistema, conciliaciones post-cierre). Se aplican las siguientes reglas:

- **Fecha retroactiva:** permitida sin restricción de rango mínimo. El usuario puede ingresar
  cualquier fecha en el pasado.
- **Fecha futura:** no permitida. El sistema valida que la fecha/hora no supere la fecha/hora
  actual del servidor al momento de guardar el movimiento. Si la fecha es futura, el sistema
  muestra error de validación y no crea el movimiento.
- **Auditoría:** cuando la fecha/hora ingresada difiere en más de 1 hora de la fecha/hora actual
  del servidor, el sistema registra automáticamente en el movimiento un campo de auditoría
  con la fecha/hora real de creación del registro (timestamp de servidor). Este campo es visible
  en el historial de movimientos con la etiqueta "Registrado el [timestamp]" para distinguirlo
  de la fecha/hora del evento.

### DD-02 — Criterio de disparo de advertencia "datos no conciliados"

La advertencia de datos no conciliados se dispara en el panel de estado general cuando se cumple
cualquiera de las siguientes condiciones:

1. Existen movimientos creados con fecha/hora retroactiva (diferencia > 1 hora entre fecha del
   evento y timestamp de creación) en las últimas 24 horas, indicando una conciliación reciente
   pendiente de validación.
2. El stock actual calculado de al menos un producto activo resulta negativo (stock < 0), lo que
   indica una excepción de despacho en rojo sin reposición posterior.

La advertencia aparece como un banner prominente antes de los KPIs con el texto:
"Advertencia: el inventario puede contener datos no conciliados. Verifique el historial antes de
tomar decisiones."

### DD-03 — Navegación al historial de excepciones para Diana

El historial de excepciones es accesible desde una vista dedicada en el menú principal de
navegación, etiquetada "Historial de Excepciones". Esta vista centraliza todas las salidas
registradas como excepción (despacho en rojo / override), independientemente del producto.
Adicionalmente, el historial de movimientos de cada producto también marca visualmente las
excepciones dentro de su propio historial (con indicador visual), permitiendo rastrear una
excepción específica desde el contexto de un producto concreto.

---

## Resumen de cobertura

| Actor (ID) | Escenarios camino feliz | Escenarios caso de borde |
|------------|------------------------|--------------------------|
| [AC-01] Andrés Mora — Almacenista | 5 (SC-01 a SC-05) | 6 (SE-01 a SE-06) |
| [AC-02] Diana Vargas — Jefa de Compras | 7 (SC-06 a SC-12) | 5 (SE-07 a SE-11) |
| [AC-03] Luis Pedraza — Gerente General | 2 (SC-13 a SC-14) | 2 (SE-12 a SE-13) |

---

## Feature: AC-01 — Andrés Mora (Almacenista)

### Escenarios de Camino Feliz

#### SC-01 — Registrar entrada de mercancía al almacén
**Fuente:** CF-01
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que existe un producto activo en el catálogo con stock actual conocido
  Y Andrés tiene mercancía para registrar como entrada (compra a proveedor o devolución de cliente)
Cuando Andrés registra una entrada seleccionando el producto, el tipo de movimiento (compra o devolución), la cantidad, la fecha/hora, y opcionalmente una referencia y una nota
Entonces el stock actual del producto aumenta en la cantidad indicada
  Y el movimiento queda registrado en el historial de movimientos como inmutable
  Y si el nuevo stock actual supera el stock mínimo del producto y existía una alerta activa, la alerta se resuelve automáticamente
```

#### SC-02 — Registrar salida de mercancía del almacén
**Fuente:** CF-02
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que existe un producto activo en el catálogo con stock actual mayor que 0
  Y la cantidad a despachar no supera el stock actual disponible del producto
Cuando Andrés registra una salida seleccionando el producto, el tipo de movimiento (despacho o pérdida/daño), la cantidad, la fecha/hora, y opcionalmente una referencia y una nota
Entonces el stock actual del producto disminuye en la cantidad indicada
  Y el movimiento queda registrado en el historial de movimientos como inmutable
  Y si el nuevo stock actual llega o cae por debajo del stock mínimo del producto, se genera una alerta de stock bajo automáticamente
```

#### SC-03 — Corregir un movimiento erróneo mediante contra-movimiento
**Fuente:** CF-03
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que existe un movimiento previo con datos incorrectos en el historial de movimientos de un producto
Cuando Andrés registra un contra-movimiento opuesto al movimiento erróneo con una nota explicativa obligatoria
Entonces el historial de movimientos muestra el movimiento original y el contra-movimiento vinculados y visibles
  Y el stock actual del producto queda corregido con el efecto neto de ambos movimientos
  Y ningún movimiento previo es eliminado ni editado
```

#### SC-04 — Ver alertas de stock bajo al ingresar al sistema
**Fuente:** CF-04
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que existen una o más alertas de stock bajo activas en el sistema
Cuando Andrés accede a la pantalla principal del sistema
Entonces la pantalla principal muestra todas las alertas de stock bajo activas en ese momento
  Y Andrés no necesita navegar a ninguna sección adicional para verlas
```

#### SC-05 — Consultar historial de movimientos de un producto
**Fuente:** CF-05
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que un producto tiene movimientos registrados en su historial
Cuando Andrés selecciona el producto y accede a su historial de movimientos
Entonces se muestra el historial completo de movimientos del producto en orden cronológico descendente
  Y cada movimiento muestra su fecha/hora, tipo, cantidad, referencia (si existe) y nota (si existe)
  Y los movimientos registrados como excepción (despacho en rojo) están marcados visualmente como tales
```

---

### Escenarios de Caso de Borde

#### SE-01 — Intento de salida que supera el stock actual disponible
**Fuente:** CB-01
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que el stock actual de un producto es menor que la cantidad que Andrés intenta despachar
Cuando Andrés intenta registrar una salida con esa cantidad
Entonces el sistema bloquea la acción
  Y muestra un mensaje de error indicando el stock actual disponible del producto
  Y ofrece a Andrés la opción de cancelar la operación o forzar la salida mediante override
```

#### SE-02 — Forzar salida con stock insuficiente mediante override (excepción)
**Fuente:** CB-02
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que el sistema bloqueó una salida por stock insuficiente y Andrés elige forzar la salida mediante override
Cuando Andrés ingresa la nota obligatoria de motivo y confirma la operación
Entonces el sistema registra la salida como excepción (despacho en rojo) en el historial de movimientos
  Y la excepción queda registrada en el historial de excepciones consultable por Diana
  Y el stock actual del producto puede quedar en negativo
```

#### SE-03 — Ingreso de movimientos retroactivos tras caída del sistema igual o mayor a 1 día
**Fuente:** CB-03
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que el sistema estuvo caído durante 1 día o más y hay movimientos que ocurrieron en papel durante la interrupción
Cuando Andrés ingresa los movimientos pendientes con la fecha/hora retroactiva del momento en que ocurrieron
Entonces los movimientos quedan registrados con la fecha/hora del evento indicada por Andrés
  Y el sistema registra automáticamente en cada movimiento un campo de auditoría con el timestamp real de creación
  Y el historial de movimientos queda completo sin huecos
```

#### SE-04 — Salida que deja el stock exactamente en 0
**Fuente:** CB-04
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que el stock actual de un producto es igual a la cantidad que Andrés quiere despachar
  Y el stock mínimo configurado del producto es mayor que 0
Cuando Andrés registra la salida
Entonces la salida se registra exitosamente
  Y el stock actual del producto queda en 0
  Y el sistema genera automáticamente una alerta de stock bajo para ese producto
```

#### SE-05 — Intento de registrar entrada a producto discontinuado
**Fuente:** CB-05
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que un producto tiene estado discontinuado en el catálogo
Cuando Andrés intenta registrar una entrada para ese producto
Entonces el sistema bloquea la acción
  Y muestra un mensaje explicativo indicando que el producto está discontinuado y no puede recibir nuevas entradas
  Y no se crea ningún movimiento
```

#### SE-06 — Intento de registrar un movimiento con fecha/hora futura
**Fuente:** CB-06 (derivado de DD-01)
**Actor:** AC-01 — Andrés Mora (Almacenista)

```gherkin
Dado que Andrés está registrando un movimiento de entrada o salida
Cuando Andrés ingresa una fecha/hora que supera la fecha/hora actual del servidor
Entonces el sistema muestra un error de validación indicando que la fecha/hora no puede ser futura
  Y no crea el movimiento
```

---

## Feature: AC-02 — Diana Vargas (Jefa de Compras)

### Escenarios de Camino Feliz

#### SC-06 — Consultar inventario en tiempo real desde cualquier dispositivo
**Fuente:** CF-06
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que Diana accede al sistema desde su laptop (en la oficina o fuera de ella) usando la interfaz web responsive
Cuando Diana navega al panel de estado general o a la lista de productos
Entonces el sistema muestra el stock actual de todos los productos calculado a partir del historial de movimientos
  Y el dato de stock actual es consistente con el que ve Andrés en tiempo real
  Y no requiere ninguna actualización manual
```

#### SC-07 — Ver detalle de alerta activa con contexto de consumo de 30 días
**Fuente:** CF-07
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que existe al menos una alerta de stock bajo activa en el sistema
Cuando Diana selecciona una alerta activa
Entonces la vista de la alerta muestra el stock actual del producto
  Y muestra el stock mínimo configurado del producto
  Y muestra los movimientos de los últimos 30 días para ese producto
```

#### SC-08 — Configurar stock mínimo de un producto en autoservicio
**Fuente:** CF-08
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que Diana accede al catálogo de productos y localiza un producto activo
Cuando Diana modifica el stock mínimo del producto directamente en la interfaz sin intervención de IT
Entonces el stock mínimo del producto queda actualizado con el nuevo valor
  Y si el nuevo stock mínimo genera o resuelve una alerta activa, el sistema recalcula el estado de alerta automáticamente
```

#### SC-09 — Consultar historial de excepciones para detectar patrones de reorden
**Fuente:** CF-09
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que existen salidas registradas como excepción (despacho en rojo) en el sistema
Cuando Diana accede al historial de excepciones desde la vista dedicada "Historial de Excepciones" en el menú principal
Entonces el sistema muestra el listado de todas las salidas registradas como excepción
  Y cada excepción muestra su nota de motivo, la fecha/hora del evento y el producto al que corresponde
```

#### SC-10 — Generar orden de reposición desde una alerta activa
**Fuente:** CF-10
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que existe una alerta activa para un producto que requiere reposición
  Y el producto tiene al menos un proveedor asociado
Cuando Diana genera una orden de reposición desde la alerta activa
Entonces el sistema crea la orden con el producto y el proveedor pre-llenados
  Y la cantidad sugerida se calcula como (stock objetivo − stock actual) del producto
  Y la cantidad sugerida es ajustable por Diana antes de confirmar
  Y la orden queda en estado Borrador
```

#### SC-11 — Confirmar una orden de reposición en estado Borrador
**Fuente:** CF-11
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que existe una orden de reposición en estado Borrador
  Y la cantidad de la orden es mayor que 0
Cuando Diana revisa y confirma la orden
Entonces la orden de reposición pasa a estado Confirmada
```

#### SC-12 — Registrar recepción de mercancía de una orden Confirmada
**Fuente:** CF-12
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que existe una orden de reposición en estado Confirmada
  Y la mercancía correspondiente ha llegado al almacén
Cuando Diana marca la orden como Recibida
Entonces el sistema crea automáticamente un movimiento de entrada por la cantidad de la orden para el producto correspondiente
  Y la orden pasa a estado Recibida
  Y si existía una alerta activa para ese producto, el sistema reevalúa el estado de alerta automáticamente
```

---

### Escenarios de Caso de Borde

#### SE-07 — Modificar stock mínimo mientras existe alerta activa para ese producto
**Fuente:** CB-07
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que un producto tiene una alerta de stock bajo activa
Cuando Diana modifica el stock mínimo del producto
Entonces si el nuevo stock mínimo es menor o igual al stock actual del producto, la alerta se resuelve automáticamente
  Y si el nuevo stock mínimo es mayor al stock actual del producto, la alerta persiste
```

#### SE-08 — Configurar stock mínimo de un producto en 0
**Fuente:** CB-08
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que Diana está modificando el stock mínimo de un producto
Cuando Diana ingresa el valor 0 como nuevo stock mínimo
Entonces el sistema muestra una advertencia en la interfaz indicando que con stock mínimo = 0 el producto nunca generará alerta de stock bajo
  Y si Diana confirma, el stock mínimo queda guardado en 0
  Y el producto no generará ninguna alerta de stock bajo aunque su stock actual llegue a 0
```

#### SE-09 — Intentar confirmar orden de reposición con cantidad 0
**Fuente:** CB-09
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que existe una orden de reposición en estado Borrador
  Y Diana ha ajustado la cantidad de la orden a 0
Cuando Diana intenta confirmar la orden
Entonces el sistema muestra un error de validación indicando que la cantidad debe ser mayor que 0
  Y la orden permanece en estado Borrador
```

#### SE-10 — Discontinuar producto con orden de reposición Confirmada pendiente
**Fuente:** CB-10
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que un producto activo tiene una orden de reposición en estado Confirmada pendiente
Cuando Diana intenta discontinuar ese producto
Entonces el sistema muestra una advertencia indicando que existe una orden Confirmada pendiente para ese producto
  Y si Diana confirma la discontinuación, el producto pasa a estado discontinuado
  Y la orden de reposición permanece en estado Confirmada sin mecanismo de cancelación en v1 (la cancelación está fuera de scope; Diana debe gestionarla fuera del sistema)
```

#### SE-11 — Acceder al historial de excepciones desde el contexto de un producto
**Fuente:** CB-11 (derivado de DD-03)
**Actor:** AC-02 — Diana Vargas (Jefa de Compras)

```gherkin
Dado que Diana está revisando el historial de movimientos de un producto específico
  Y ese producto tiene movimientos registrados como excepción (despacho en rojo)
Cuando Diana revisa el historial de movimientos del producto
Entonces los movimientos registrados como excepción aparecen con un indicador visual que los distingue del resto
  Y Diana puede identificar cada excepción directamente en el historial del producto sin necesidad de cambiar de vista
```

---

## Feature: AC-03 — Luis Pedraza (Gerente General)

### Escenarios de Camino Feliz

#### SC-13 — Ver KPIs ejecutivos en el panel de estado general
**Fuente:** CF-13
**Actor:** AC-03 — Luis Pedraza (Gerente General)

```gherkin
Dado que Luis accede al sistema para revisar el estado del inventario o reportar a socios
Cuando Luis accede al panel de estado general
Entonces la pantalla muestra de forma prominente el KPI de productos con alerta de stock bajo activa en ese momento
  Y muestra el KPI de despachos registrados como excepción (despacho en rojo) en el mes en curso
  Y ambos KPIs son conteos numéricos sin valoración monetaria
```

#### SC-14 — Filtrar tabla de productos por categoría o estado en el panel
**Fuente:** CF-14
**Actor:** AC-03 — Luis Pedraza (Gerente General)

```gherkin
Dado que Luis está en el panel de estado general con la tabla de todos los productos visible
Cuando Luis aplica un filtro por categoría o por estado del producto (OK / ALERTA / SIN MOVIMIENTOS)
Entonces la tabla muestra únicamente los productos que corresponden al filtro seleccionado
```

---

### Escenarios de Caso de Borde

#### SE-12 — Ver advertencia de datos no conciliados antes de los KPIs
**Fuente:** CB-12
**Actor:** AC-03 — Luis Pedraza (Gerente General)

```gherkin
Dado que el sistema detecta la condición de datos no conciliados (movimientos retroactivos registrados en las últimas 24 horas, o al menos un producto con stock actual negativo)
Cuando Luis accede al panel de estado general
Entonces el sistema muestra un banner prominente antes de los KPIs con el texto: "Advertencia: el inventario puede contener datos no conciliados. Verifique el historial antes de tomar decisiones."
  Y los KPIs se muestran a continuación del banner, no en lugar de él
  Y Luis no ve los KPIs sin haber visto primero la advertencia
```

#### SE-13 — Producto sin movimientos aparece en el panel de estado general
**Fuente:** CB-13
**Actor:** AC-03 — Luis Pedraza (Gerente General)

```gherkin
Dado que existe un producto activo que no tiene ningún movimiento registrado
Cuando Luis visualiza la tabla de productos en el panel de estado general
Entonces ese producto aparece en la tabla con estado SIN MOVIMIENTOS (distinto del estado ALERTA)
  Y su stock actual se muestra como 0
  Y el producto no genera una alerta de stock bajo por este estado
```

---

## Feature: AC-04 — Vendedores

Sin escenarios especificables en v1. Los vendedores no tienen interfaz propia en esta versión.
Su necesidad de disponibilidad de stock (OV-13) es satisfecha indirectamente por el sistema
mediante las alertas y órdenes de reposición gestionadas por los actores AC-01 y AC-02.
