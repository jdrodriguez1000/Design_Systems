# Domain Glossary — 010 Discovery
Fecha: 2026-05-31

## Términos del Dominio

| Término | Definición acordada | Sinónimos a evitar | Actor que lo usa |
|---------|--------------------|--------------------|-----------------|
| Entrada de comida | Registro individual de un alimento o bebida consumido, compuesto por un nombre descriptivo libre y un valor de calorías en kilocalorías. | "alimento", "ítem", "food log", "producto" | Sofía Martínez |
| Meta calórica | Número de kilocalorías diarias que Sofía establece como objetivo de consumo máximo para el día. Es editable manualmente en cualquier momento. | "límite diario", "objetivo", "target", "goal" | Sofía Martínez |
| Total consumido | Suma acumulada de todas las kilocalorías de las entradas de comida registradas durante el día actual. Se muestra en todo momento. | "calorías ingeridas", "consumo del día", "total del día" | Sofía Martínez |
| Saldo restante | Diferencia entre la meta calórica y el total consumido, cuando la diferencia es positiva (aún dentro de la meta). Indica cuántas kilocalorías quedan disponibles para el día. | "calorías disponibles", "calorías que quedan", "margen" | Sofía Martínez |
| Exceso calórico | Diferencia entre el total consumido y la meta calórica, cuando el total supera la meta. Se muestra como texto "Excediste por X kcal" con cambio de color visual (rojo/naranja). | "sobreconsumo", "pasarse", "calorías extras" | Sofía Martínez |
| Reinicio diario | Evento automático que ocurre a medianoche y borra las entradas del día anterior, iniciando el contador del día nuevo desde cero. No requiere intervención de la usuaria. | "reset", "limpieza diaria", "nuevo día" | Sofía Martínez |
| Historial del día | Conjunto completo de entradas de comida registradas durante el día actual. Persiste en el navegador ante un cierre accidental. Se borra únicamente con el reinicio diario a medianoche. | "lista del día", "registros del día", "log diario" | Sofía Martínez |
| Valor inválido | Cualquier entrada en el campo de calorías que no sea un número entero mayor a cero. Incluye: texto no numérico, cero, números negativos. El sistema rechaza la entrada y muestra mensaje de error. | "valor incorrecto", "dato incorrecto", "error de entrada" | Sofía Martínez |

## Abreviaturas

| Abreviatura | Expansión | Contexto de uso |
|-------------|-----------|-----------------|
| kcal | kilocalorías | Unidad de medida usada en todo el sistema para expresar el valor energético de las entradas de comida, la meta calórica y el exceso calórico. |
