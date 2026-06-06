# BDD Feature Files — 020 Specification
Fecha: 2026-05-31
Estado: DRAFT
Basado en: specification/spec_analysis_report.md

## Resumen de cobertura

| Actor (ID) | Escenarios camino feliz | Escenarios caso de borde |
|------------|------------------------|--------------------------|
| [AC-01] Sofía Martínez | 8 | 10 |

---

## Feature: Sofía Martínez — Registro y seguimiento calórico diario

### Escenarios de Camino Feliz

#### SC-01 — Registrar entrada de comida con nombre y calorías válidos
**Fuente:** CF-01
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que la aplicación está abierta y Sofía Martínez acaba de comer
Cuando Sofía Martínez ingresa el nombre de la entrada de comida y un número entero mayor a cero en el campo de calorías (kcal) y confirma
Entonces la entrada de comida queda guardada en el historial del día
  Y el total consumido se actualiza sumando las calorías de la nueva entrada de comida
```

#### SC-02 — Consultar el total consumido del día
**Fuente:** CF-02
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que el historial del día tiene al menos una entrada de comida registrada
Cuando Sofía Martínez consulta la pantalla principal
Entonces el total consumido se muestra actualizado con la suma de las calorías de todas las entradas de comida del historial del día
```

#### SC-03 — Ver saldo restante cuando el total consumido es menor que la meta calórica
**Fuente:** CF-03
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que hay una meta calórica configurada
  Y el total consumido es menor que la meta calórica
Cuando Sofía Martínez consulta la pantalla principal
Entonces se muestra el saldo restante (meta calórica menos total consumido)
  Y se muestra el total consumido
```

#### SC-04 — Ver exceso calórico cuando el total consumido supera la meta calórica
**Fuente:** CF-04
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que hay una meta calórica configurada
  Y el total consumido supera la meta calórica
Cuando Sofía Martínez consulta la pantalla principal
Entonces se muestra el texto "Excediste por X kcal" con el valor del exceso calórico
  Y la indicación visual cambia a color rojo o naranja
```

#### SC-05 — Establecer o modificar la meta calórica
**Fuente:** CF-05
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que Sofía Martínez desea establecer o modificar su meta calórica
Cuando Sofía Martínez ingresa un nuevo valor de meta calórica (número entero mayor a cero) y confirma
Entonces la meta calórica queda actualizada con el nuevo valor
  Y el saldo restante o el exceso calórico se recalcula inmediatamente usando la nueva meta calórica
```

#### SC-06 — Eliminar una entrada de comida incorrecta del historial del día
**Fuente:** CF-06
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que el historial del día tiene una entrada de comida con calorías incorrectas
Cuando Sofía Martínez elimina esa entrada de comida
Entonces la entrada de comida se elimina del historial del día
  Y el total consumido se recalcula descontando las calorías de la entrada eliminada
```

#### SC-07 — Recuperar el historial del día tras cerrar y reabrir el navegador
**Fuente:** CF-07
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que Sofía Martínez ha registrado entradas de comida en el historial del día
  Y cierra el navegador o la pestaña
Cuando Sofía Martínez vuelve a abrir la aplicación en el mismo navegador
Entonces el historial del día con todas las entradas de comida previamente registradas está disponible
  Y el total consumido es el mismo que antes del cierre del navegador
```

#### SC-08 — Reinicio diario automático a medianoche
**Fuente:** CF-08
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que es medianoche
  Y el historial del día tiene entradas de comida del día anterior registradas
Cuando el sistema ejecuta el reinicio diario automáticamente
Entonces el historial del día se borra completamente
  Y el total consumido vuelve a cero
  Y la meta calórica configurada se mantiene para el nuevo día
```

---

### Escenarios de Caso de Borde

#### SE-01 — Rechazo de calorías con valor cero al registrar entrada de comida
**Fuente:** CB-01
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que Sofía Martínez está intentando registrar una entrada de comida
  Y el campo de calorías contiene el valor cero
Cuando Sofía Martínez confirma el ingreso
Entonces la entrada de comida no se guarda en el historial del día
  Y se muestra el mensaje "ingresa un número mayor a cero" junto al campo de calorías
```

#### SE-02 — Rechazo de calorías con valor negativo al registrar entrada de comida
**Fuente:** CB-02
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que Sofía Martínez está intentando registrar una entrada de comida
  Y el campo de calorías contiene un número negativo
Cuando Sofía Martínez confirma el ingreso
Entonces la entrada de comida no se guarda en el historial del día
  Y se muestra el mensaje "ingresa un número mayor a cero" junto al campo de calorías
```

#### SE-03 — Rechazo de calorías con texto no numérico al registrar entrada de comida
**Fuente:** CB-03
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que Sofía Martínez está intentando registrar una entrada de comida
  Y el campo de calorías contiene texto no numérico
Cuando Sofía Martínez confirma el ingreso
Entonces la entrada de comida no se guarda en el historial del día
  Y se muestra el mensaje "ingresa un número mayor a cero" junto al campo de calorías
```

#### SE-04 — Rechazo de calorías con campo vacío al registrar entrada de comida
**Fuente:** CB-04
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que Sofía Martínez está intentando registrar una entrada de comida
  Y el campo de calorías está vacío
Cuando Sofía Martínez confirma el ingreso
Entonces la entrada de comida no se guarda en el historial del día
  Y se muestra el mensaje "ingresa un número mayor a cero" junto al campo de calorías
```

#### SE-05 — Consulta de pantalla principal sin meta calórica configurada
**Fuente:** CB-05
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que no hay meta calórica configurada
Cuando Sofía Martínez consulta la pantalla principal
Entonces se muestra el total consumido
  Y se muestra un recordatorio visible de que no hay meta calórica configurada
  Y no se muestra el saldo restante
```

#### SE-06 — Cierre del navegador con entrada de comida en progreso
**Fuente:** CB-06
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que Sofía Martínez tiene el formulario de ingreso de una entrada de comida a medio completar
  Y el historial del día tiene entradas de comida previamente guardadas
Cuando el navegador se cierra
Entonces la entrada de comida en progreso puede perderse
  Y las entradas de comida ya guardadas en el historial del día persisten
```

#### SE-07 — Meta calórica borrada o vaciada sin ingresar una nueva
**Fuente:** CB-07
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que Sofía Martínez borra o vacía el campo de meta calórica sin ingresar un nuevo valor
Cuando la aplicación procesa el cambio
Entonces se muestra el total consumido
  Y se muestra un recordatorio visible de que no hay meta calórica configurada
  Y no se muestra el saldo restante ni el exceso calórico
```

#### SE-08 — Corrección de entrada de comida mediante eliminación y reingreso
**Fuente:** CB-08
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que el historial del día tiene una entrada de comida con calorías incorrectas
Cuando Sofía Martínez elimina la entrada de comida incorrecta
  Y Sofía Martínez ingresa una nueva entrada de comida con los valores correctos
Entonces la entrada incorrecta se elimina del historial del día
  Y el total consumido se recalcula sin las calorías de la entrada eliminada
  Y la nueva entrada de comida con los valores correctos queda guardada en el historial del día
```

#### SE-09 — Total consumido exactamente igual a la meta calórica
**Fuente:** CB-09
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que hay una meta calórica configurada
  Y el total consumido es exactamente igual a la meta calórica
Cuando Sofía Martínez consulta la pantalla principal
Entonces se muestra el saldo restante con valor de cero kcal
```

> **Resuelto por cliente 2026-05-31:** total = meta se trata como **dentro de la meta**. Saldo restante: 0 kcal. El estado de exceso calórico SOLO se activa cuando total > meta (estrictamente mayor). Sin color de alerta.

#### SE-10 — Historial del día vacío al iniciar el día o tras el reinicio diario
**Fuente:** CB-10
**Actor:** AC-01 — Sofía Martínez

```gherkin
Dado que el historial del día no tiene ninguna entrada de comida registrada
Cuando Sofía Martínez consulta la pantalla principal
Entonces el total consumido se muestra como cero kcal
  Y si hay meta calórica configurada, el saldo restante es igual al valor de la meta calórica
```
