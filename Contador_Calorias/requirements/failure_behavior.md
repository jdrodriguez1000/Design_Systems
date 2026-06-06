# Failure Behavior — 010 Discovery
Fecha: 2026-05-31
Destino: input directo para Error & Exception Policy del 020 Specification Harness

## Escenarios de Fallo por Actor

### Sofía Martínez (A-01)

| ID    | Escenario | Causa probable | Comportamiento esperado | Prioridad |
|-------|-----------|---------------|------------------------|-----------|
| SF-01 | Cierre accidental del navegador a mitad del día | La usuaria cierra la pestaña, el navegador o el dispositivo se apaga inesperadamente antes de que termine el día. [Inferencia: causa derivada del escenario declarado por S-01] | El historial de comidas ya registradas en el día debe persistir y estar disponible al reabrir la página. La entrada en progreso al momento del cierre puede perderse. | alta |
| SF-02 | Error de registro: valor incorrecto ingresado intencionalmente | La usuaria ingresó calorías equivocadas y lo advierte después de guardar la entrada. [Inferencia: error humano de digitación al momento del registro] | La usuaria puede eliminar la entrada incorrecta y reingresarla con el valor correcto. La edición directa de una entrada existente es el comportamiento preferido pero no es bloqueante si no se implementa en la primera versión. | media |
| SF-03 | Valor inválido en el campo de calorías al intentar registrar una entrada | La usuaria ingresó cero, un número negativo, texto no numérico o dejó el campo vacío. [Inferencia: error de digitación o incomprensión del campo] | No guardar la entrada. Mostrar mensaje de error: "ingresa un número mayor a cero". No inferir un valor por defecto ni usar el último valor ingresado. | alta |
| SF-04 | Meta calórica no configurada al intentar consultar el saldo | La usuaria abrió la app por primera vez sin haber establecido una meta, o la borró sin ingresar una nueva. [Inferencia: estado inicial de la app o borrado accidental de la meta] | No mostrar el saldo restante (no hay meta contra la cual calcular). Sí mostrar el total de calorías consumidas hasta el momento. Mostrar un recordatorio visible de que no hay meta calórica configurada. | media |

## Escenarios Globales (no atribuibles a un actor específico)

No se identificaron escenarios de fallo globales en el analysis_report. El sistema tiene un único actor.

## Items sin Respuesta del Cliente

| Escenario | Pregunta pendiente | Impacto en el 020 |
|-----------|-------------------|--------------------|
| Funcionamiento sin conexión a internet | No se definió si la app debe funcionar sin conexión. La usuaria accede desde celular inmediatamente después de comer, pero no declaró conectividad offline como requisito. (V-03 del analysis_report — bajo impacto, no bloqueante) | El 020 deberá decidir si la arquitectura asume conexión disponible o si se requiere funcionamiento offline. No bloquea la especificación de comportamientos funcionales. |
