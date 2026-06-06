# Mantenibilidad — Contador de Calorías

## Por qué importa la mantenibilidad en un sistema pequeño

El código que parece "tan simple que no necesita estructura" es el que más rápido se convierte en deuda técnica. Una app personal de calorías puede empezar como 50 líneas en un `<script>` inline — y 6 meses después, si Sofía pide editar entradas y añadir categorías, esas 50 líneas son imposibles de modificar sin romper algo.

---

## Principios SOLID aplicados

### S — Responsabilidad Única (el más crítico aquí)

Cada módulo debe tener una sola razón para cambiar.

**Aplicación concreta**:
- `StorageRepository`: cambia solo si cambia cómo se persisten los datos (ej: migrar de localStorage a IndexedDB)
- `CalorieCalculator`: cambia solo si cambia la lógica de negocio (ej: añadir macronutrientes en el futuro)
- `Validator`: cambia solo si cambian las reglas de validación
- `UIRenderer`: cambia solo si cambia cómo se presenta la información

**Señal de alerta**: si modificar el formato de `localStorage` requiere tocar el código de validación de entradas, hay una violación de responsabilidad única.

### O — Abierto/Cerrado

Las Display Strategies del documento de patrones son un ejemplo directo: para añadir un nuevo estado de visualización (ej: "Estás en tu exacta meta calórica" con color verde especial), se agrega una nueva estrategia sin modificar las existentes.

### L — Sustitución de Liskov

Aplica si se usan abstracciones. En JavaScript vanilla: si `StorageRepository` se reemplaza por `MockStorageRepository` en tests (que usa un objeto en memoria en lugar de `localStorage`), el resto del código debe funcionar igual sin saber el cambio.

### I — Segregación de Interfaces

Más relevante en TypeScript. Si se define la interfaz de `StorageRepository`, no debe obligar a implementar métodos que el módulo que la usa no necesita.

### D — Inversión de Dependencias

La lógica de negocio (`CalorieCalculator`) no debe importar directamente `localStorage`. Debe recibir `StorageRepository` como dependencia — esto facilita los tests.

---

## Separación de concerns

**Estructura recomendada de módulos**:

```
src/
├── storage.js      → Solo habla con localStorage
├── calculator.js   → Solo hace cálculos (funciones puras, sin efectos)
├── validator.js    → Solo valida entradas
├── renderer.js     → Solo actualiza el DOM
└── app.js          → Orquestador: conecta los módulos anteriores
```

**Por qué funciones puras en `calculator.js`**:

```javascript
// Función pura — mismo input, mismo output, sin efectos secundarios
function calculateTotal(entries) {
  return entries.reduce((sum, e) => sum + e.calorias, 0);
}

function calculateBalance(total, goal) {
  if (goal === null) return { type: 'no-goal' };
  if (total > goal) return { type: 'exceeded', amount: total - goal };
  return { type: 'within', remaining: goal - total };
}
```

Las funciones puras son triviales de testear, fáciles de razonar, y no tienen sorpresas.

---

## Deuda técnica anticipada

| Atajo | Deuda que genera | Cómo mitigarlo |
|-------|-----------------|----------------|
| Todo en `index.html` con un `<script>` inline | Imposible testear unitariamente; los cambios son frágiles | Separar en módulos desde el inicio |
| Usar `innerHTML` para renderizar entradas | Vulnerabilidad XSS si el nombre contiene HTML | Usar `textContent` o `createElement` siempre |
| El ID de cada entrada es solo `Date.now()` | Riesgo teórico de colisión si se agrega rápido | Aceptable para V1; documentar el riesgo |
| Sin types en JavaScript | Errores de tipo solo se descubren en runtime | Usar JSDoc o migrar a TypeScript en V2 |
| Lógica de reinicio diario sin test | Difícil de detectar bugs en el comportamiento de medianoche | Testear con fechas mockeadas |

---

## Estrategia de versionado

Para una app personal sin releases formales, el versionado es simple:
- **V1**: funcionalidad completa de los 20 ACP documentados
- **V2**: añadir DF-01 (edición directa de entradas) si Sofía lo pide
- **V3+**: DF-02, DF-03, DF-04 según demanda

El código debe estar estructurado de forma que añadir edición directa (DF-01) solo requiera:
1. Añadir un campo editable en el renderer para cada entrada
2. Añadir una función `updateEntry(id, newData)` en el módulo de lógica
3. No tocar la validación ni el storage

---

## Code smells a vigilar en este sistema

| Code Smell | Señal | Solución |
|-----------|-------|---------|
| **God Object** | `app.js` tiene 500+ líneas y hace todo | Extraer en módulos por responsabilidad |
| **Primitive Obsession** | `calorias` se pasa como string en algunos lugares y como number en otros | Normalizar a number al leer de localStorage, siempre |
| **Magic Numbers** | `if (value <= 0)` disperso en varios lugares | Constante `MIN_CALORIES = 1` o función `isValidCalories(v)` |
| **Direct DOM manipulation en lógica** | `document.getElementById` llamado desde `calculator.js` | El cálculo nunca debe tocar el DOM |
| **Commented-out code** | Bloques de código comentados que "por si acaso" | Si no se usa, se borra; el historial está en git |
