# Patrones de Diseño — Contador de Calorías

## Introducción

Para un sistema tan pequeño, aplicar demasiados patrones es contraproducente. Los patrones son soluciones a problemas recurrentes — si el problema no existe, el patrón no se necesita. Aquí solo documentamos los 3-4 patrones que resuelven problemas reales de este sistema.

---

## 1. Repository Pattern (Estructural / Arquitectónico)

### El problema que resuelve
Sin este patrón, la lógica de negocio (calcular saldo, validar entradas) estaría mezclada con las llamadas a `localStorage`. Si en el futuro se migra a un backend, habría que reescribir código de negocio para desacoplar los accesos a datos.

### Cómo se aplica aquí
Se crea un módulo `StorageRepository` que es la única parte del sistema que conoce `localStorage`. El resto de la app habla con el repositorio, no con `localStorage` directamente.

```javascript
// StorageRepository — única interfaz con localStorage
const StorageRepository = {
  getEntries() {
    return JSON.parse(localStorage.getItem('calorie_tracker_entries') || '[]');
  },
  saveEntries(entries) {
    localStorage.setItem('calorie_tracker_entries', JSON.stringify(entries));
  },
  getGoal() {
    const val = localStorage.getItem('calorie_tracker_goal');
    return val ? parseInt(val, 10) : null;
  },
  saveGoal(goal) {
    localStorage.setItem('calorie_tracker_goal', goal ? String(goal) : '');
  },
  getLastDate() {
    return localStorage.getItem('calorie_tracker_date');
  },
  saveDate(dateStr) {
    localStorage.setItem('calorie_tracker_date', dateStr);
  }
};
```

**Ventaja concreta**: si mañana se migra a IndexedDB o a un API backend, solo cambia `StorageRepository` — el resto del código no se toca.

---

## 2. Strategy Pattern (Comportamiento)

### El problema que resuelve
La pantalla principal tiene 3 estados distintos según la relación entre `totalConsumed` y `goal`:
- **Sin meta**: mostrar total + recordatorio
- **Dentro de meta** (total ≤ goal): mostrar saldo restante
- **Exceso** (total > goal): mostrar exceso en rojo/naranja

Sin un patrón, esto se convierte en un bloque `if/else if/else` que crece con el tiempo y mezcla lógica de cálculo con lógica de presentación.

### Cómo se aplica aquí

```javascript
const DisplayStrategies = {
  noGoal: (total) => ({
    message: `Total consumido: ${total} kcal`,
    reminder: 'No has configurado una meta calórica.',
    color: 'default'
  }),
  withinGoal: (total, goal) => ({
    message: `Total: ${total} kcal`,
    balance: `Te quedan ${goal - total} kcal`,
    color: 'green'
  }),
  exceeded: (total, goal) => ({
    message: `Total: ${total} kcal`,
    balance: `Excediste por ${total - goal} kcal`,
    color: 'red'
  })
};

function getDisplayStrategy(total, goal) {
  if (goal === null) return DisplayStrategies.noGoal(total);
  if (total > goal) return DisplayStrategies.exceeded(total, goal);
  return DisplayStrategies.withinGoal(total, goal);
}
```

Este patrón encapsula cada estado de visualización como una función pura y testeable por separado.

---

## 3. Observer / Reactive State Pattern (Comportamiento)

### El problema que resuelve
Cuando Sofía agrega una entrada, elimina una entrada, o cambia la meta calórica, **múltiples partes de la UI deben actualizarse**: el total consumido, el saldo/exceso, el historial. Sin coordinación, se corre el riesgo de que alguna parte quede desactualizada.

### Cómo se aplica aquí
Se mantiene un **estado central** (objeto JavaScript) y una función `render()` que siempre lee ese estado y actualiza toda la UI. Cada acción del usuario muta el estado y llama a `render()`.

```javascript
// Estado central — fuente de verdad
let appState = {
  entries: [],  // desde StorageRepository
  goal: null    // desde StorageRepository
};

// Toda actualización de UI pasa por aquí
function render(state) {
  const total = state.entries.reduce((sum, e) => sum + e.calorias, 0);
  const display = getDisplayStrategy(total, state.goal);
  updateDOM(display);
  renderEntriesList(state.entries);
}

// Acción: agregar entrada
function addEntry(nombre, calorias) {
  appState.entries.push({ id: Date.now(), nombre, calorias });
  StorageRepository.saveEntries(appState.entries);
  render(appState);  // toda la UI se actualiza desde un solo lugar
}
```

Este patrón es la base de frameworks como React y Vue — aquí se implementa en vanilla JS de manera simplificada.

---

## 4. Guard Clause (Creacional / Estructural — buena práctica)

### El problema que resuelve
La validación de entradas (SE-01 a SE-04) podría escribirse como un if gigante. Las Guard Clauses son el patrón de salida temprana que hace el código de validación legible.

```javascript
function validateCalories(input) {
  if (input === '' || input === null) return 'ingresa un número mayor a cero';
  const num = Number(input);
  if (isNaN(num)) return 'ingresa un número mayor a cero';
  if (!Number.isInteger(num)) return 'ingresa un número mayor a cero';
  if (num <= 0) return 'ingresa un número mayor a cero';
  return null; // válido
}
```

Cada condición inválida retorna inmediatamente con el mensaje exacto especificado en EP-03. Si llega al final, la entrada es válida.

---

## Patrones que NO se aplican aquí (y por qué)

| Patrón | Por qué no aplica |
|--------|-------------------|
| **Singleton** | No hay instancias múltiples que coordinar; todo vive en un módulo |
| **Factory** | Solo se crea un tipo de entidad simple; no hay jerarquía de objetos |
| **CQRS** | Separar lectura/escritura tiene sentido con alta carga concurrente; aquí hay un solo usuario |
| **Event Sourcing** | Reconstruir estado desde eventos tiene sentido para auditoría o sistemas distribuidos; aquí `localStorage` es suficiente |
| **Saga** | Para orquestar transacciones distribuidas; no aplica a una SPA sin backend |
