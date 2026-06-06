# Scope Boundaries — 010 Discovery
Fecha: 2026-05-31

## Qué NO hará el sistema en esta etapa

| ID    | Exclusión | Razón | Fuente |
|-------|-----------|-------|--------|
| EX-01 | No incluirá catálogo de alimentos ni base de datos nutricional | La usuaria registra nombre y calorías manualmente. No expresó necesidad de búsqueda ni autocompletado de alimentos. | Transcript — decisión explícita de S-01 |
| EX-02 | No calculará ni mostrará macronutrientes (proteínas, grasas, carbohidratos) | La usuaria solo mencionó calorías como unidad de seguimiento. Ningún objetivo de valor hace referencia a otros nutrientes. | Transcript — scope acordado con S-01 |
| EX-03 | No tendrá historial de días anteriores ni reportes semanales o mensuales | El sistema muestra únicamente el día actual. Las entradas se eliminan con el reinicio diario a medianoche. Sofía no solicitó acceso a días pasados. | Transcript — decisión explícita de S-01; límite implícito del scope acordado |
| EX-04 | No tendrá múltiples usuarios ni sistema de autenticación | La app es de uso personal exclusivo de Sofía Martínez. No se mencionó compartir datos ni control de acceso. | Transcript — descripción del actor A-01 |
| EX-05 | No se declara funcionamiento offline como requisito | La necesidad de funcionar sin conexión a internet no fue expresada por la usuaria. El acceso desde celular no implica requisito offline. | V-03 del analysis_report — no bloqueante |
| EX-06 | No tendrá sincronización entre dispositivos | La usuaria accede desde celular (principal) y computadora (secundario), pero no solicitó que los datos del día estén disponibles en ambos dispositivos simultáneamente. | Límite implícito del scope; no mencionado por S-01 |

## Qué queda diferido (posibles futuras fases)

| ID    | Capacidad diferida | Condición para incluir |
|-------|--------------------|----------------------|
| DF-01 | Edición directa de entradas registradas (modificar nombre o calorías sin eliminar y reingresar) | Sofía la marcó como preferida pero no bloqueante. Puede incorporarse si la implementación de eliminación+reingreso resulta incómoda en uso real. |
| DF-02 | Historial de días anteriores | Requeriría redefinir el modelo de datos y la política de reinicio diario. Sujeto a que la usuaria lo solicite tras usar la versión inicial. |
| DF-03 | Catálogo o autocompletado de alimentos comunes | Requiere base de datos nutricional. Sujeto a que la usuaria lo solicite por repetición frecuente de los mismos alimentos. |
| DF-04 | Sincronización entre celular y computadora | Requiere backend con cuenta de usuario. Sujeto a que la brecha de datos entre dispositivos resulte problemática en uso real. |

## Restricciones activas

| Tipo | Restricción | Impacto |
|------|-------------|---------|
| scope | Solo el día actual es visible; no hay acceso a días pasados desde la interfaz | Simplifica el modelo de datos pero impide comparaciones históricas |
| uso | Un único usuario (Sofía Martínez); sin autenticación requerida | Elimina complejidad de gestión de acceso pero acopla el diseño a uso personal |
| entrada de datos | Las calorías se ingresan manualmente como número entero mayor a cero; no se acepta ningún otro formato | Requiere validación de entrada clara y mensaje de error comprensible |
| tiempo | El reinicio diario es automático a medianoche; no hay opción de reinicio manual ni de ajuste de zona horaria declarado | El 020 deberá definir qué zona horaria usa el sistema para determinar la medianoche |
