# Data Contracts — 020 Specification
Fecha: 2026-05-31
Estado: DRAFT
Basado en: specification/spec_analysis_report.md

## Entidades y sus Contratos

### Entidad EN-01 — Entrada de comida

**Descripción:** Registro individual de un alimento o bebida consumido por Sofía Martínez. Se compone de un nombre descriptivo libre y un valor de calorías expresado en kcal. Es la unidad básica del historial del día.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| nombre | texto | texto libre, sin restricción de formato | sí | No puede estar vacío. No existe catálogo de referencia (EX-01). | SC-01, SE-08 |
| calorías | número entero | número entero mayor a cero (kcal) | sí | Debe ser un número entero mayor a cero. Se rechaza: cero, número negativo, texto no numérico, campo vacío. Valor inválido genera mensaje "ingresa un número mayor a cero". | SC-01, SE-01, SE-02, SE-03, SE-04, SE-08 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-01 | Una entrada de comida solo se guarda en el historial del día si el campo de calorías contiene un número entero mayor a cero. | alta |
| RN-02 | No se admiten macronutrientes ni ningún atributo nutricional adicional al valor de calorías en kcal. | alta |
| RN-03 | La entrada de comida no es editable directamente una vez guardada. Para corregirla, Sofía Martínez debe eliminarla y reingresarla. La edición directa queda diferida a DF-01. | media |

---

### Entidad EN-02 — Meta calórica

**Descripción:** Número de kcal diarias que Sofía Martínez establece como objetivo de consumo máximo para el día. Es editable manualmente en cualquier momento. Su presencia es opcional; si no está configurada, el sistema muestra un recordatorio visible.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| valor | número entero | número entero mayor a cero (kcal) | no | Debe ser un número entero mayor a cero si está configurada. El sistema opera sin meta calórica configurada, mostrando recordatorio. | SC-03, SC-04, SC-05, SE-05, SE-07, SE-09, SE-10 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-04 | Si la meta calórica no está configurada, el sistema no muestra saldo restante ni exceso calórico, pero sí muestra el total consumido y un recordatorio visible. | alta |
| RN-05 | Al actualizar la meta calórica, el saldo restante o el exceso calórico se recalcula inmediatamente con el nuevo valor. | alta |
| RN-06 | La meta calórica persiste tras el reinicio diario; no se borra con el historial del día. | alta |

---

### Entidad EN-03 — Total consumido

**Descripción:** Suma acumulada de todas las kcal de las entradas de comida registradas durante el día actual. Es un valor calculado, no ingresado por Sofía Martínez. Se muestra en todo momento en la pantalla principal.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| valor | número entero calculado | número entero mayor o igual a cero (kcal) | sí (calculado) | Es la suma de las calorías de todas las entradas de comida del historial del día. Si el historial del día está vacío, el valor es cero. | SC-01, SC-02, SC-03, SC-04, SC-06, SC-07, SC-08, SE-10 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-07 | El total consumido se recalcula cada vez que se agrega o elimina una entrada de comida del historial del día. | alta |
| RN-08 | El total consumido vuelve a cero tras el reinicio diario. | alta |

---

### Entidad EN-04 — Saldo restante

**Descripción:** Diferencia entre la meta calórica y el total consumido cuando la diferencia es positiva. Indica cuántas kcal quedan disponibles para el día. Es un valor calculado; solo se muestra cuando la meta calórica está configurada y el total consumido es menor que la meta calórica.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| valor | número entero calculado | número entero mayor o igual a cero (kcal) | no (calculado, condicional) | Solo se calcula y muestra cuando: (1) la meta calórica está configurada y (2) el total consumido es menor que la meta calórica. Si el total consumido es igual a la meta calórica, el saldo restante es cero. | SC-03, SE-09, SE-10 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-09 | Saldo restante = meta calórica − total consumido. Solo visible cuando el resultado es mayor o igual a cero y la meta calórica está configurada. | alta |
| RN-10 | Si la meta calórica no está configurada, el saldo restante no se muestra. | alta |

---

### Entidad EN-05 — Exceso calórico

**Descripción:** Diferencia entre el total consumido y la meta calórica cuando el total supera la meta. Se muestra como texto "Excediste por X kcal" con cambio de color visual a rojo o naranja. Es un valor calculado; solo se muestra cuando el total consumido supera la meta calórica.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| valor | número entero calculado | número entero mayor a cero (kcal) | no (calculado, condicional) | Solo se calcula y muestra cuando el total consumido supera la meta calórica. La presentación usa el texto exacto "Excediste por X kcal" con indicación visual en color rojo o naranja. | SC-04 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-11 | Exceso calórico = total consumido − meta calórica. Solo visible cuando el total consumido es estrictamente mayor que la meta calórica. | alta |
| RN-12 | El exceso calórico requiere que la meta calórica esté configurada. Si no hay meta calórica, no se muestra exceso calórico. | alta |
| RN-13 | El exceso calórico se muestra con texto "Excediste por X kcal" y cambio de color visual (rojo o naranja) en la pantalla principal. | alta |

---

### Entidad EN-06 — Historial del día

**Descripción:** Conjunto completo de entradas de comida registradas durante el día actual. Persiste en el navegador ante un cierre accidental. Se borra únicamente con el reinicio diario a medianoche. Solo contiene las entradas del día actual; los días anteriores no son accesibles (EX-03).

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| entradas | colección de entradas de comida | lista ordenada de entradas de comida (EN-01) | no (puede estar vacío) | Puede contener cero o más entradas de comida. Solo entradas del día actual. | SC-01, SC-02, SC-06, SC-07, SC-08, SE-06, SE-08, SE-10 |
| fecha del día | fecha | día actual | sí | Corresponde al día actual. Se renueva con el reinicio diario. | SC-08, SE-10 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-14 | El historial del día persiste en el navegador (mediante almacenamiento local) ante cierres accidentales. | alta |
| RN-15 | El historial del día solo contiene entradas de comida del día actual. No se accede a días anteriores (EX-03). | alta |
| RN-16 | El historial del día se borra completamente con el reinicio diario a medianoche. | alta |
| RN-17 | Una entrada de comida en proceso de ingreso (formulario a medio completar) al momento del cierre del navegador puede perderse; solo las entradas ya guardadas persisten. | alta |

---

### Entidad EN-07 — Reinicio diario

**Descripción:** Evento automático que ocurre a medianoche y borra el historial del día anterior, iniciando el nuevo día desde cero. No requiere intervención de Sofía Martínez. La meta calórica no se borra con el reinicio diario.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| hora de ejecución | hora del día | medianoche (00:00) | sí | El reinicio ocurre automáticamente a medianoche. No hay opción de reinicio manual. | SC-08 |
| zona horaria | zona horaria local del dispositivo del usuario (browser / sistema operativo) | — | sí | El reinicio ocurre a medianoche según la zona horaria local del dispositivo. No se configura en servidor. | SC-08 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-18 | El reinicio diario es automático; no existe opción de reinicio manual. | alta |
| RN-19 | El reinicio diario borra el historial del día pero preserva la meta calórica. | alta |
| RN-20 | La zona horaria que determina la medianoche es la zona horaria local del dispositivo del usuario (browser/sistema operativo). El reinicio no depende de configuración en servidor. | alta |

---

## Relaciones entre Entidades

| ID    | Entidad A | Relación | Entidad B | Cardinalidad | Restricción de negocio |
|-------|-----------|----------|-----------|--------------|------------------------|
| RE-01 | Historial del día | contiene | Entrada de comida | 0..N | El historial del día puede tener cero o más entradas de comida. Se vacía con el reinicio diario. |
| RE-02 | Total consumido | se calcula a partir de | Historial del día | 1..1 | El total consumido es la suma de todas las calorías de las entradas del historial del día actual. |
| RE-03 | Saldo restante | se calcula a partir de | Meta calórica | 1..1 | El saldo restante requiere que la meta calórica esté configurada y que el total consumido sea menor que la meta. |
| RE-04 | Saldo restante | se calcula a partir de | Total consumido | 1..1 | Saldo restante = meta calórica − total consumido, solo cuando el resultado es positivo o cero. |
| RE-05 | Exceso calórico | se calcula a partir de | Total consumido | 1..1 | Exceso calórico = total consumido − meta calórica, solo cuando el total consumido supera la meta. |
| RE-06 | Exceso calórico | se calcula a partir de | Meta calórica | 1..1 | El exceso calórico requiere que la meta calórica esté configurada. |
| RE-07 | Reinicio diario | borra | Historial del día | 1..1 | El reinicio diario ocurre a medianoche y vacía completamente el historial del día. |
