# Error & Exception Policy — 020 Specification
Fecha: 2026-05-31
Estado: DRAFT
Basado en: specification/spec_analysis_report.md

## Políticas por Actor

### Actor AC-01 — Sofía Martínez

| ID     | Escenario de error (EE-xx) | Causa | Mensaje al usuario | Reintento | Bloqueo | Acción alternativa | Escenario BDD relacionado |
|--------|---------------------------|-------|-------------------|-----------|---------|-------------------|--------------------------|
| EP-01  | EE-01 — Cierre accidental del navegador a mitad del día | El navegador o la pestaña se cierra mientras hay una sesión activa con entradas de comida registradas. | (ningún mensaje — el sistema recupera el historial automáticamente al reabrir) | no | no | El historial del día persiste automáticamente en el almacenamiento local del navegador. Al reabrir la aplicación en el mismo navegador, las entradas de comida guardadas están disponibles. La entrada en progreso al momento del cierre puede perderse. | SE-06, SC-07 |
| EP-02  | EE-02 — Valor de calorías incorrecto ya guardado en una entrada de comida | Sofía Martínez ingresó un valor de calorías incorrecto y lo confirmó. La entrada de comida incorrecta ya está en el historial del día. | (ningún mensaje automático — Sofía Martínez identifica el error visualmente en el historial del día) | no | no | Sofía Martínez puede eliminar la entrada de comida incorrecta del historial del día. Puede luego ingresar una nueva entrada de comida con los valores correctos. La edición directa de entradas queda diferida a DF-01. | SC-06, SE-08 |
| EP-03  | EE-03 — Valor inválido en el campo de calorías al intentar registrar una entrada de comida | El campo de calorías contiene un valor que no es un número entero mayor a cero: cero, número negativo, texto no numérico o campo vacío. | "ingresa un número mayor a cero" | no | sí (bloquea el guardado de la entrada de comida hasta que se corrija el campo) | El sistema no guarda la entrada de comida. Muestra el mensaje junto al campo de calorías. Sofía Martínez debe corregir el valor antes de poder confirmar el ingreso. | SE-01, SE-02, SE-03, SE-04 |
| EP-04  | EE-04 — Meta calórica no configurada o borrada al consultar el estado calórico | Sofía Martínez consulta la pantalla principal sin haber configurado una meta calórica, o la ha borrado o vaciado sin ingresar un nuevo valor. | "No has configurado una meta calórica. Ingresa tu meta para ver tu saldo restante." | no | no | El sistema muestra el total consumido normalmente. No muestra saldo restante ni exceso calórico. Muestra el recordatorio visible en la pantalla principal hasta que Sofía Martínez configure una meta calórica. | SE-05, SE-07 |

---

## Políticas Globales

| ID     | Escenario de error (EE-xx) | Causa | Mensaje al usuario | Reintento | Bloqueo | Acción alternativa |
|--------|---------------------------|-------|-------------------|-----------|---------|-------------------|
| EP-05  | EE-05 — Funcionamiento sin conexión a internet | La aplicación intenta operar sin conexión a internet disponible. | (no aplica — la aplicación asume conexión disponible; el funcionamiento offline está fuera del scope del sistema) | no aplica | no aplica | No se implementa lógica offline. No se genera política de excepción para ausencia de conexión. Este escenario está excluido por resolución del governor (EX-05). |

---

## Resoluciones de ítems PENDIENTE del 010

| Ítem original (EE-xx) | Pregunta que estaba pendiente | Resolución del governor | Política aplicada (EP-xx) |
|-----------------------|------------------------------|------------------------|--------------------------|
| EE-05 (V-03 del failure_behavior.md) | ¿Debe la aplicación funcionar sin conexión a internet? El ítem estaba marcado como PENDIENTE en failure_behavior.md. | V-03: la app asume conexión disponible. El funcionamiento offline está fuera del scope del sistema. No se implementa lógica offline ni política de excepción para este caso. | EP-05 |
