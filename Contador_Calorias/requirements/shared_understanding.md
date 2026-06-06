# Shared Understanding Document — 010 Discovery
Fecha: 2026-05-31
Estado: APROBADO

## Propósito del Sistema

"Mi Contador de Calorías" es una aplicación web personal diseñada para una sola persona: Sofía Martínez. Su propósito es eliminar la necesidad de hacer cuentas mentales o en papel para saber cuántas calorías lleva consumidas en el día y cuántas le quedan para alcanzar su meta diaria.

El problema que resuelve es concreto: Sofía quiere registrar lo que come a lo largo del día ingresando un nombre y el número de calorías de cada comida, y en todo momento quiere ver de un vistazo si va bien o si ya se excedió de su meta. La app no busca ser un sistema de nutrición completo ni conectarse a bases de datos de alimentos; el registro es simple, manual y directo.

La app funciona dentro del día: a medianoche se reinicia sola, borrando los registros del día anterior y dejando el contador en cero para el nuevo día. Si Sofía cierra el navegador por accidente, sus registros del día no se pierden; están ahí cuando vuelve a abrir la página. La meta calórica la establece ella misma y puede cambiarla cuando quiera.

## Actores y sus Necesidades

### Sofía Martínez (A-01)
- **Descripción:** Única usuaria de la aplicación. Uso personal, no técnico. Accede principalmente desde su celular (inmediatamente después de comer) y ocasionalmente desde computadora.
- **Necesidades principales:**
  - Ver en todo momento cuántas calorías lleva consumidas en el día sin hacer cuentas a mano (OV-01).
  - Ver cuántas calorías le quedan para alcanzar su meta diaria, o cuánto se excedió si ya superó la meta (OV-02).
  - Registrar cada comida ingresando un nombre descriptivo y la cantidad de calorías (OV-03).
  - Establecer y editar libremente su meta calórica diaria cuando lo desee (OV-04).
  - Que los datos del día persistan si cierra el navegador accidentalmente (OV-05).
  - Poder corregir una entrada incorrecta eliminándola y reingresándola; la edición directa es preferible pero no es un bloqueo (OV-06).
  - Ver claramente cuando se excede de su meta: texto "Excediste por X kcal" con cambio de color a rojo o naranja (OV-07).
  - Que el contador del día se reinicie solo a medianoche sin que ella tenga que hacer nada (OV-08).

## Lo que el Sistema Hace

1. El sistema permite registrar una entrada de comida ingresando un nombre libre y un número de calorías mayor a cero.
2. El sistema muestra en todo momento el total de calorías consumidas en el día actual, sumando todas las entradas registradas.
3. El sistema permite establecer y modificar una meta calórica diaria en cualquier momento.
4. El sistema muestra el saldo de calorías restantes (meta menos total consumido) cuando hay una meta configurada y el total no la ha superado.
5. El sistema muestra el mensaje "Excediste por X kcal" con cambio de color a rojo o naranja cuando el total consumido supera la meta calórica.
6. El sistema muestra un recordatorio visible cuando no hay meta calórica configurada, sin mostrar saldo restante pero sí mostrando el total consumido.
7. El sistema rechaza entradas de calorías con valor inválido (cero, negativo, texto no numérico) y muestra el mensaje "ingresa un número mayor a cero".
8. El sistema permite eliminar una entrada de comida registrada para corregir un error.
9. El sistema persiste el historial de entradas del día en el navegador, de modo que si la usuaria cierra la pestaña o el navegador y lo vuelve a abrir, los datos del día están disponibles.
10. El sistema reinicia automáticamente el contador a medianoche, borrando las entradas del día anterior e iniciando el nuevo día desde cero, sin intervención de la usuaria.

## Contradicciones Resueltas

No se detectaron contradicciones durante el proceso de discovery. El sistema cuenta con un único stakeholder entrevistado (S-01).

Las siguientes ambigüedades y vacíos fueron resueltos durante la iteración 2 del análisis:

| Tema | Resolución acordada |
|------|---------------------|
| Reinicio diario: ¿automático o manual? (AM-01) | Reinicio automático a medianoche, sin intervención de la usuaria. |
| Comportamiento ante valor de calorías inválido (V-01) | No guardar la entrada. Mostrar mensaje "ingresa un número mayor a cero". |
| Comportamiento cuando no hay meta configurada (V-02) | Mostrar total consumido y un recordatorio visible. No mostrar saldo restante. |

## Aprobación del Cliente

Estado: APROBADO
Fecha de aprobación: 2026-05-31
Registro: 2026-05-31T22:41:51Z — Sofía Martínez aprobó formalmente el Shared Understanding Document con "aprobados". Aprobación CP-04 registrada.
