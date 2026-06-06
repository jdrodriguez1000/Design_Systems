# Product Acceptance Criteria — 020 Specification
Fecha: 2026-05-31
Estado: DRAFT
Basado en: specification/spec_analysis_report.md + specification/bdd_features.md

## Criterios de Aceptación por Actor

### Actor AC-01 — Sofía Martínez

| ID     | Criterio | Escenario BDD | Condición de cumplimiento | Condición de fallo |
|--------|----------|---------------|--------------------------|-------------------|
| ACP-01 | Al confirmar el ingreso de una entrada de comida con nombre y calorías válidos, la entrada queda guardada y el total consumido se actualiza. | SC-01 | La entrada de comida aparece en el historial del día y el total consumido refleja la suma incluyendo las calorías recién ingresadas. | La entrada no aparece en el historial del día, o el total consumido no refleja el valor ingresado. |
| ACP-02 | La pantalla principal muestra el total consumido actualizado con la suma de todas las entradas de comida del historial del día. | SC-02 | El valor del total consumido en pantalla es igual a la suma de las calorías de todas las entradas de comida del historial del día. | El total consumido muestra un valor distinto a la suma real de las entradas registradas. |
| ACP-03 | Cuando el total consumido es menor que la meta calórica, la pantalla principal muestra el saldo restante. | SC-03 | El saldo restante mostrado es igual a la meta calórica menos el total consumido, y el valor es correcto numéricamente. | El saldo restante no se muestra, o el valor calculado es incorrecto. |
| ACP-04 | Cuando el total consumido supera la meta calórica, la pantalla principal muestra el texto "Excediste por X kcal" con cambio de color a rojo o naranja. | SC-04 | El texto "Excediste por X kcal" aparece con el valor correcto del exceso calórico y el elemento visual está en color rojo o naranja. | El mensaje no aparece, el valor del exceso calórico es incorrecto, o no hay cambio de color. |
| ACP-05 | Al confirmar una nueva meta calórica, el valor queda actualizado y el saldo restante o exceso calórico se recalcula inmediatamente. | SC-05 | La meta calórica en pantalla refleja el nuevo valor ingresado, y el saldo restante o exceso calórico se recalcula sin requerir ninguna acción adicional de Sofía Martínez. | La meta calórica no se actualiza, o el recálculo del saldo restante o exceso calórico no ocurre inmediatamente. |
| ACP-06 | Al eliminar una entrada de comida, esta desaparece del historial del día y el total consumido se recalcula correctamente. | SC-06 | La entrada de comida eliminada no aparece en el historial del día y el total consumido refleja la suma de las entradas restantes. | La entrada eliminada sigue visible en el historial, o el total consumido no se ajusta tras la eliminación. |
| ACP-07 | Al reabrir la aplicación en el mismo navegador después de cerrarla, el historial del día y el total consumido son los mismos que antes del cierre. | SC-07 | Todas las entradas de comida guardadas antes del cierre del navegador están presentes en el historial del día, y el total consumido coincide con el valor previo al cierre. | Falta alguna entrada de comida previamente guardada, o el total consumido es distinto al que había antes del cierre. |
| ACP-08 | A medianoche el sistema borra automáticamente el historial del día, reinicia el total consumido a cero y preserva la meta calórica. | SC-08 | Después del reinicio diario: el historial del día está vacío, el total consumido es cero, y la meta calórica tiene el mismo valor que tenía antes del reinicio. | El historial del día no se borra, el total consumido no vuelve a cero, o la meta calórica se pierde tras el reinicio. |
| ACP-09 | El sistema rechaza el ingreso de una entrada de comida cuando el campo de calorías contiene el valor cero y muestra el mensaje "ingresa un número mayor a cero". | SE-01 | La entrada de comida no se guarda y aparece el mensaje "ingresa un número mayor a cero" junto al campo de calorías. | La entrada se guarda con valor cero, o el mensaje de error no aparece, o aparece un mensaje diferente. |
| ACP-10 | El sistema rechaza el ingreso de una entrada de comida cuando el campo de calorías contiene un número negativo y muestra el mensaje "ingresa un número mayor a cero". | SE-02 | La entrada de comida no se guarda y aparece el mensaje "ingresa un número mayor a cero" junto al campo de calorías. | La entrada se guarda con valor negativo, o el mensaje de error no aparece, o aparece un mensaje diferente. |
| ACP-11 | El sistema rechaza el ingreso de una entrada de comida cuando el campo de calorías contiene texto no numérico y muestra el mensaje "ingresa un número mayor a cero". | SE-03 | La entrada de comida no se guarda y aparece el mensaje "ingresa un número mayor a cero" junto al campo de calorías. | La entrada se guarda, o el mensaje de error no aparece, o aparece un mensaje diferente. |
| ACP-12 | El sistema rechaza el ingreso de una entrada de comida cuando el campo de calorías está vacío y muestra el mensaje "ingresa un número mayor a cero". | SE-04 | La entrada de comida no se guarda y aparece el mensaje "ingresa un número mayor a cero" junto al campo de calorías. | La entrada se guarda con campo vacío, o el mensaje de error no aparece, o aparece un mensaje diferente. |
| ACP-13 | Cuando no hay meta calórica configurada, la pantalla principal muestra el total consumido y un recordatorio visible de que no hay meta calórica configurada, sin mostrar saldo restante. | SE-05 | El total consumido es visible, el recordatorio de meta calórica no configurada aparece en pantalla, y el saldo restante no se muestra. | El saldo restante aparece sin meta calórica configurada, o el recordatorio no es visible, o el total consumido no se muestra. |
| ACP-14 | Si el navegador se cierra mientras hay una entrada de comida en progreso, las entradas ya guardadas del historial del día persisten al reabrir la aplicación. | SE-06 | Todas las entradas de comida previamente guardadas aparecen en el historial del día al reabrir la aplicación. La entrada en progreso al momento del cierre puede no estar presente. | Entradas de comida ya guardadas antes del cierre no aparecen en el historial del día al reabrir. |
| ACP-15 | Al borrar o vaciar la meta calórica sin ingresar un nuevo valor, la pantalla principal muestra el total consumido y un recordatorio visible de que no hay meta calórica configurada, sin mostrar saldo restante ni exceso calórico. | SE-07 | El total consumido es visible, el recordatorio de meta calórica no configurada aparece en pantalla, y ni el saldo restante ni el exceso calórico se muestran. | El saldo restante o el exceso calórico siguen mostrándose sin meta calórica, o el recordatorio no aparece. |
| ACP-16 | Al eliminar una entrada de comida incorrecta y reingresar una nueva con los valores correctos, el historial del día y el total consumido reflejan el estado correcto. | SE-08 | La entrada incorrecta no aparece en el historial del día, la nueva entrada con valores correctos sí aparece, y el total consumido refleja la suma correcta. | La entrada incorrecta sigue visible, la nueva entrada no se guarda, o el total consumido no es correcto. |
| ACP-17 | Cuando el total consumido es exactamente igual a la meta calórica, el sistema muestra saldo restante de cero kcal y no activa el estado de exceso calórico. | SE-09 | El saldo restante se muestra con valor cero kcal y no se activa el estado de exceso calórico (total = meta no es exceso). | El saldo restante no muestra cero kcal, o se activa el estado de exceso calórico cuando total = meta, o se muestra color de alerta. |
| ACP-18 | Cuando el historial del día está vacío, el total consumido se muestra como cero kcal; si hay meta calórica configurada, el saldo restante es igual a la meta calórica. | SE-10 | El total consumido muestra cero kcal. Si la meta calórica está configurada, el saldo restante muestra el mismo valor que la meta calórica. | El total consumido no muestra cero, o el saldo restante no coincide con la meta calórica cuando el historial está vacío. |

---

## Criterios de Aceptación Globales

| ID     | Criterio | Escenario BDD | Condición de cumplimiento | Condición de fallo |
|--------|----------|---------------|--------------------------|-------------------|
| ACP-19 | El valor inválido en el campo de calorías siempre produce el mensaje "ingresa un número mayor a cero" y nunca guarda la entrada de comida. | SE-01, SE-02, SE-03, SE-04 | En todos los casos de valor inválido (cero, negativo, texto no numérico, vacío), el mensaje es exactamente "ingresa un número mayor a cero" y no existe ninguna entrada guardada para ese intento. | El mensaje varía según el tipo de valor inválido, o algún tipo de valor inválido permite guardar la entrada de comida. |
| ACP-20 | La persistencia del historial del día en el navegador es efectiva: ninguna entrada de comida ya guardada se pierde por el cierre del navegador. | SC-07, SE-06 | Tras reabrir la aplicación en el mismo navegador, todas las entradas de comida que estaban guardadas antes del cierre están presentes. | Alguna entrada guardada previamente no aparece tras reabrir la aplicación. |

---

## Trazabilidad inversa

| Escenario BDD | Criterio de aceptación |
|---------------|------------------------|
| SC-01 | ACP-01 |
| SC-02 | ACP-02 |
| SC-03 | ACP-03 |
| SC-04 | ACP-04 |
| SC-05 | ACP-05 |
| SC-06 | ACP-06 |
| SC-07 | ACP-07, ACP-20 |
| SC-08 | ACP-08 |
| SE-01 | ACP-09, ACP-19 |
| SE-02 | ACP-10, ACP-19 |
| SE-03 | ACP-11, ACP-19 |
| SE-04 | ACP-12, ACP-19 |
| SE-05 | ACP-13 |
| SE-06 | ACP-14, ACP-20 |
| SE-07 | ACP-15 |
| SE-08 | ACP-16 |
| SE-09 | ACP-17 |
| SE-10 | ACP-18 |
