# Error & Exception Policy — 020 Specification
Fecha: 2026-06-01
Estado: DRAFT
Basado en: specification/spec_analysis_report.md (Error & Exception Mapping EE-01 a EE-13)

---

## Propósito

Este documento define la política formal de manejo de errores y excepciones del Sistema de
Inventario y Alertas de Stock. Cada política cubre: qué ocurre, cómo responde el sistema,
qué ve el usuario, y qué queda registrado. Las políticas resuelven los 13 ítems del
`failure_behavior.md` y las 3 decisiones de diseño (DD-01, DD-02, DD-03) delegadas a esta
fase de especificación.

---

## Sección 1 — Políticas de Error en el Módulo de Movimientos

### EE-01 — Movimiento con Datos Incorrectos (Error de Registro)

**Fuente:** failure_behavior.md F-01
**Escenario BDD:** SC-03

**Situación:** Un movimiento fue registrado con datos incorrectos (cantidad errónea, tipo
equivocado, producto equivocado).

**Política:**
- El historial de movimientos es inmutable. Ningún movimiento puede eliminarse ni editarse
  directamente bajo ninguna circunstancia.
- La corrección se realiza **exclusivamente** mediante un contra-movimiento: un nuevo movimiento
  de tipo opuesto que revierte el efecto del movimiento erróneo.
- El contra-movimiento requiere nota obligatoria con la explicación del error.
- El historial del producto muestra el movimiento original y el contra-movimiento como registros
  vinculados y visibles, dejando trazabilidad completa.

**Respuesta del sistema:**
- No existe botón ni opción de "editar movimiento" ni "eliminar movimiento" en la interfaz.
- El usuario registra el contra-movimiento desde el formulario de nuevo movimiento, seleccionando
  el movimiento de origen.
- El stock actual queda corregido con el efecto neto de ambos registros.

**Registro:** Ambos movimientos permanecen en el historial de forma permanente.

---

### EE-02 — Salida que Supera el Stock Disponible (Override)

**Fuente:** failure_behavior.md F-02
**Escenario BDD:** SE-01, SE-02

**Situación:** Andrés intenta registrar una salida cuya cantidad supera el stock actual del
producto.

**Política:**
- **Bloqueo por defecto:** el sistema bloquea la acción y muestra un mensaje de error que
  incluye el stock actual disponible del producto.
- **Override autorizado:** el sistema ofrece la opción de forzar la salida mediante override.
  Esta opción está disponible para permitir despachos cuando se sabe que la mercancía llegará
  ese día (~2-3 veces por mes según stakeholders).
- El override requiere **nota obligatoria** con el motivo. El formulario no puede enviarse sin ella.
- Al confirmar el override, la salida se registra como excepción (despacho en rojo) con flag
  excepción = sí.
- El stock actual puede quedar en negativo tras el override.

**Respuesta del sistema al bloqueo:**
- Mensaje: "Stock insuficiente. Stock actual disponible: [N unidades]. Puedes cancelar o
  forzar la salida indicando el motivo."
- Dos opciones disponibles: [Cancelar] y [Forzar salida con nota].

**Respuesta del sistema al override:**
- El movimiento queda registrado con flag excepción = sí y nota de excepción.
- Aparece en el historial de movimientos del producto con indicador visual de excepción.
- Aparece en el historial de excepciones consultable por Diana.

**Registro:** Movimiento con flag excepción = sí, nota de excepción, timestamp de creación.

---

### EE-03 — Caída del Sistema Inferior a 1 Hora

**Fuente:** failure_behavior.md F-03

**Situación:** El sistema no está disponible por menos de 1 hora.

**Política:**
- Esta situación es una interrupción operativa tolerada, no un fallo del software.
- No requiere ninguna acción técnica especial dentro del sistema.
- El negocio opera temporalmente en papel o hoja de cálculo.
- Al recuperarse el sistema, Andrés ingresa los movimientos pendientes con fecha/hora retroactiva
  (ver EE-04 para la política de movimientos retroactivos).

**Registro:** Los movimientos ingresados retroactivamente quedarán marcados con flag auditoría
retroactiva = sí si la diferencia entre la fecha del evento y el timestamp de creación es
mayor a 1 hora (ver EE-04).

---

### EE-04 — Caída del Sistema Igual o Mayor a 1 Día (Movimientos Retroactivos)

**Fuente:** failure_behavior.md F-04
**Decisión de diseño resuelta:** DD-01
**Escenario BDD:** SE-03, SE-06

**Situación:** El sistema estuvo inactivo 1 día o más. Al recuperarse, existen movimientos
que ocurrieron en papel durante la interrupción y deben ingresarse con fecha/hora retroactiva.

**Política:**
- La fecha/hora del evento en un movimiento **es editable por el usuario**. El usuario puede
  ingresar cualquier fecha pasada sin límite mínimo de antigüedad.
- **Fecha futura:** prohibida. El sistema valida que la fecha/hora ingresada no supere la
  fecha/hora actual del servidor en el momento de guardar. Si la fecha es futura, el sistema
  muestra error de validación y no crea el movimiento.
- **Auditoría de retroactividad:** cuando la diferencia entre la fecha/hora del evento
  ingresada por el usuario y el timestamp de creación del registro (generado automáticamente
  por el servidor) es mayor a 1 hora, el sistema activa automáticamente el flag de auditoría
  retroactiva = sí. Este flag no es editable por el usuario.
- El timestamp de creación es siempre la fecha/hora real del servidor al guardar; no es
  editable.
- El campo de auditoría retroactiva es visible en el historial de movimientos con la etiqueta
  "Registrado el [timestamp de creación]" para distinguirlo de la fecha/hora del evento.

**Respuesta del sistema a fecha futura:**
- Mensaje: "La fecha/hora del evento no puede ser futura. Por favor ingresa la fecha y hora
  correctas."
- El movimiento no se crea.

**Respuesta del sistema a fecha retroactiva válida:**
- El movimiento se registra con la fecha/hora del evento indicada por el usuario.
- El flag auditoría retroactiva se activa automáticamente si aplica.
- El historial queda completo sin huecos.

**Registro:** Movimiento con fecha/hora del evento (ingresada por usuario), timestamp de
creación (automático del servidor), flag auditoría retroactiva (automático).

---

### EE-05 — Dato Presentado con Posible Inconsistencia (Datos No Conciliados)

**Fuente:** failure_behavior.md F-05
**Decisión de diseño resuelta:** DD-02
**Escenario BDD:** SE-12

**Situación:** El inventario calculado podría no reflejar la realidad. Luis está en el panel
ejecutivo y podría ver un KPI incorrecto con apariencia de certeza.

**Política:**
- Cuando se detecta la condición de "datos no conciliados", el sistema muestra un **banner
  prominente** antes de los KPIs en el panel de estado general.
- Texto exacto del banner: *"Advertencia: el inventario puede contener datos no conciliados.
  Verifique el historial antes de tomar decisiones."*
- Los KPIs se muestran a continuación del banner, no en lugar de él. Luis nunca ve los KPIs
  sin haber visto primero la advertencia.

**Condición técnica de disparo del banner (DD-02 resuelto):**
El banner se activa cuando se cumple **cualquiera** de las siguientes condiciones:

1. Existen movimientos con flag auditoría retroactiva = sí cuyo timestamp de creación sea
   de las últimas 24 horas (indica conciliación reciente pendiente de validación).
2. El stock actual calculado de al menos un producto activo es negativo (indica un override
   sin reposición posterior).

Si no se cumple ninguna condición, el banner no aparece y los KPIs se muestran directamente.

**Respuesta del sistema:**
- Banner visible antes de los KPIs; no bloquea la vista ni requiere confirmación del usuario
  para verlos.
- El banner desaparece automáticamente cuando ninguna condición de disparo está activa.

**Registro:** No se crea un registro adicional por el banner. Las condiciones de disparo son
evaluadas en tiempo real al cargar el panel.

---

## Sección 2 — Políticas de Casos de Borde del Catálogo y Alertas

### EE-06 — Salida que Deja el Stock Exactamente en 0

**Fuente:** failure_behavior.md B-01
**Escenario BDD:** SE-04

**Política:**
- La salida se permite sin restricción adicional.
- Después del registro, el sistema evalúa si el nuevo stock (= 0) es ≤ stock mínimo del
  producto.
- Si el stock mínimo del producto es > 0: se genera automáticamente una alerta de stock bajo.
- Si el stock mínimo del producto es 0: no se genera alerta (ver EE-08).

---

### EE-07 — Producto sin Movimientos Registrados

**Fuente:** failure_behavior.md B-02
**Escenario BDD:** SE-13

**Política:**
- El producto aparece en la tabla del panel de estado general con estado **SIN MOVIMIENTOS**
  (distinto de ALERTA y de OK).
- Su stock actual se muestra como 0.
- No se genera alerta de stock bajo para este producto por este estado.
- Este estado distingue productos recién creados sin actividad de productos con historial que
  actualmente tienen stock en cero.

---

### EE-08 — Stock Mínimo Configurado en 0

**Fuente:** failure_behavior.md — Nota RN-02; risk R-03 del scope_boundaries.md
**Escenario BDD:** SE-08

**Política:**
- Un producto con stock mínimo = 0 **nunca** genera alerta de stock bajo, aunque su stock
  actual llegue a 0.
- Al configurar el stock mínimo en 0, el sistema muestra una **advertencia en la interfaz**:
  *"Con stock mínimo = 0 este producto nunca generará alerta de stock bajo."*
- La advertencia aparece antes de guardar; si Diana confirma, el valor se guarda en 0 y el
  comportamiento sin alerta queda activo.
- Esta política resuelve el riesgo R-03 identificado en el analysis_report.md: el usuario es
  informado explícitamente del efecto antes de confirmar.

---

### EE-09 — Proveedor sin Datos de Contacto Completos

**Fuente:** failure_behavior.md B-03

**Política:**
- El sistema permite crear y usar un proveedor con solo el campo nombre completado.
- Los campos teléfono y email son opcionales; no tienen validación de formato en v1.
- Una orden de reposición puede generarse aunque el proveedor no tenga datos de contacto.
  Diana gestiona el contacto fuera del sistema.

---

### EE-10 — Entrada a Producto Discontinuado

**Fuente:** failure_behavior.md B-04
**Escenario BDD:** SE-05

**Política:**
- El sistema bloquea el registro de cualquier movimiento de entrada a un producto con estado
  discontinuado.
- Muestra mensaje explicativo: *"Este producto está discontinuado y no puede recibir nuevas
  entradas."*
- No se crea el movimiento.
- El historial de movimientos del producto discontinuado se preserva y es consultable.

---

### EE-11 — Orden de Reposición con Cantidad Ajustada a 0

**Fuente:** failure_behavior.md B-05
**Escenario BDD:** SE-09

**Política:**
- El sistema no permite confirmar una orden de reposición cuya cantidad ajustada sea 0.
- Al intentar confirmar, muestra error de validación: *"La cantidad debe ser mayor que 0 para
  confirmar la orden."*
- La orden permanece en estado Borrador.

---

### EE-12 — Dos Alertas Simultáneas del Mismo Producto

**Fuente:** failure_behavior.md B-06

**Política:**
- Esta situación no ocurre por diseño del sistema.
- Solo puede existir una alerta activa por producto simultáneamente.
- Si el stock baja más mientras la alerta ya está activa, la alerta existente persiste sin
  crear una segunda alerta.
- No se requiere ningún manejo de error adicional.

---

### EE-13 — Discontinuar Producto con Orden Confirmada Pendiente

**Fuente:** failure_behavior.md B-07
**Escenario BDD:** SE-10

**Política:**
- El sistema permite discontinuar el producto, pero **muestra advertencia** antes de confirmar:
  *"Existe una orden de reposición en estado Confirmada para este producto. Si lo discontinúas,
  deberás cancelar la orden manualmente."*
- Si el usuario confirma la discontinuación, el producto pasa a estado discontinuado.
- La orden de reposición permanece en estado Confirmada; no cambia de estado automáticamente.
- La cancelación de órdenes está **fuera de scope v1**. El sistema no provee mecanismo para cancelar una orden en estado Confirmada. La orden permanece en ese estado hasta v2; Diana debe gestionar la cancelación fuera del sistema.

---

### EE-14 — Stock Mínimo Modificado con Alerta Activa Existente

**Fuente:** failure_behavior.md B-08
**Escenario BDD:** SE-07

**Política:**
- Al modificar el stock mínimo de un producto con alerta activa, el sistema reevalúa
  inmediatamente la condición de alerta:
  - Si nuevo stock mínimo ≤ stock actual: la alerta se resuelve automáticamente.
  - Si nuevo stock mínimo > stock actual: la alerta persiste.
- La reevaluación ocurre en el momento de guardar el nuevo stock mínimo, sin acción adicional
  del usuario.

---

## Sección 3 — Política de Navegación al Historial de Excepciones (DD-03)

**Decisión de diseño resuelta:** DD-03
**Escenario BDD:** SC-09, SE-11

La navegación al historial de excepciones para Diana es accesible por dos rutas complementarias:

1. **Vista dedicada en el menú principal:** existe una entrada en el menú de navegación
   principal etiquetada "Historial de Excepciones". Desde esta vista se listan todas las
   salidas registradas como excepción de todos los productos, independientemente del producto.
   Diana puede detectar patrones cross-producto desde esta vista.

2. **Indicador visual en el historial de movimientos por producto:** en la vista de historial
   de movimientos de un producto específico, los movimientos con flag excepción = sí aparecen
   con un indicador visual que los distingue del resto. Diana puede identificar excepciones
   de un producto concreto sin cambiar de vista.

Ambas rutas son complementarias y no excluyentes.

---

## Sección 4 — Restricciones de Scope Aplicadas a Esta Política

Las siguientes exclusiones del `scope_boundaries.md` afectan directamente lo que NO está
cubierto por esta política de errores y excepciones:

| Exclusión | Impacto en la política de errores |
|-----------|----------------------------------|
| EX-01 — Sin precios ni valoración monetaria | Ningún mensaje de error incluye información de costos o valores monetarios. |
| EX-03 — Sin autenticación diferenciada (roles) | La política no distingue permisos por usuario. Cualquier usuario puede realizar cualquier operación cubierta en v1. |
| EX-04 — Sin notificaciones push ni email | Las alertas y advertencias son exclusivamente en-pantalla. No se envían emails ni notificaciones externas. |
| EX-07 — Sin lotes ni fechas de vencimiento | Ningún error o excepción cubre vencimiento de stock. |
| EX-09 — Sin flujo de aprobación de órdenes | No existe estado de "pendiente de aprobación" en las órdenes de reposición; Diana confirma directamente. |

---

## Resumen de Políticas

| ID    | Ítem de origen | Tipo | Política principal |
|-------|---------------|------|--------------------|
| EE-01 | F-01 | Error de registro | Inmutabilidad total; corrección por contra-movimiento con nota obligatoria |
| EE-02 | F-02 | Excepción de salida | Bloqueo por defecto; override con nota obligatoria; registro como excepción |
| EE-03 | F-03 | Caída < 1h | Tolerada; sin acción técnica; conciliación posterior por usuario |
| EE-04 | F-04 + DD-01 | Caída ≥ 1 día; fecha futura | Retroactivo sin límite; fecha futura bloqueada; auditoría automática si diferencia > 1h |
| EE-05 | F-05 + DD-02 | Datos no conciliados | Banner antes de KPIs; disparo por retroactivos 24h o stock negativo |
| EE-06 | B-01 | Stock = 0 tras salida | Permitida; alerta si mínimo > 0 |
| EE-07 | B-02 | Producto sin movimientos | Estado SIN MOVIMIENTOS; sin alerta |
| EE-08 | RN-02 + R-03 | Stock mínimo = 0 | Advertencia UI al configurar; nunca genera alerta |
| EE-09 | B-03 | Proveedor sin datos | Solo nombre obligatorio; campos contacto opcionales |
| EE-10 | B-04 | Entrada a discontinuado | Bloqueada con mensaje explicativo |
| EE-11 | B-05 | Orden cantidad = 0 | No confirmable; error de validación |
| EE-12 | B-06 | Dos alertas simultáneas | No ocurre por diseño |
| EE-13 | B-07 | Discontinuar con orden Confirmada | Advertencia; orden Confirmada permanece sin mecanismo de cancelación en v1 (fuera de scope) |
| EE-14 | B-08 | Stock mínimo modificado con alerta | Reevaluación automática; resuelve o persiste según nuevo mínimo vs stock actual |
| Secc. 3 | DD-03 | Navegación historial excepciones | Vista dedicada en menú + indicador visual en historial por producto |
