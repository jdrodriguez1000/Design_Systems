# Estrategia de Testing — Contador de Calorías

## La pirámide de testing aplicada

```
        /\
       /E2E\          ~10% — Flujos completos de usuario
      /──────\
     /Integr. \       ~20% — Persistencia en localStorage
    /────────── \
   / Unit Tests  \    ~70% — Lógica de negocio y validación
  /______________  \
```

Para una SPA sin backend, la pirámide se invierte ligeramente en la base: **los tests unitarios de lógica pura son el mayor valor**. La lógica de negocio (calcular saldo, detectar exceso, validar entradas) es determinista y fácil de probar sin infraestructura.

---

## Qué testear obligatoriamente

### 1. Validación de entradas (SE-01 a SE-04) — Prioridad ALTA

Estos 4 escenarios tienen criterios de aceptación exactos con mensajes de error específicos. Una regresión aquí significa que Sofía puede registrar entradas inválidas sin saberlo.

```javascript
// Casos que NUNCA deben quedar sin cobertura
describe('validateCalories', () => {
  test('rechaza valor cero',          () => expect(validate(0)).toBe('ingresa un número mayor a cero'));
  test('rechaza valor negativo',      () => expect(validate(-1)).toBe('ingresa un número mayor a cero'));
  test('rechaza texto no numérico',   () => expect(validate('abc')).toBe('ingresa un número mayor a cero'));
  test('rechaza campo vacío',         () => expect(validate('')).toBe('ingresa un número mayor a cero'));
  test('acepta entero positivo',      () => expect(validate(100)).toBe(null));
  test('acepta exactamente 1',        () => expect(validate(1)).toBe(null));
});
```

### 2. Cálculos calóricos — Prioridad ALTA

Las reglas de negocio RN-07 a RN-13 son funciones matemáticas puras. Perfectas para unit tests.

```javascript
describe('calculateBalance', () => {
  test('sin meta → tipo no-goal',              () => expect(calc(500, null).type).toBe('no-goal'));
  test('total < meta → saldo correcto',        () => expect(calc(500, 1800).remaining).toBe(1300));
  test('total = meta → saldo 0, no exceso',    () => expect(calc(1800, 1800)).toEqual({ type: 'within', remaining: 0 }));
  test('total > meta → exceso correcto',       () => expect(calc(2000, 1800)).toEqual({ type: 'exceeded', amount: 200 }));
  test('historial vacío → total 0',            () => expect(sumEntries([])).toBe(0));
});
```

**Nota sobre SE-09** (total exactamente igual a la meta): este caso de borde fue explícitamente resuelto por el cliente. El test que verifica que `total === meta` NO activa el exceso calórico es obligatorio.

### 3. Reinicio diario — Prioridad ALTA

La lógica de detectar cambio de día es simple pero crítica (ACP-08). Se prueba mockeando la fecha.

```javascript
describe('shouldReset', () => {
  test('misma fecha → no reinicia',         () => expect(shouldReset('2026-06-06', '2026-06-06')).toBe(false));
  test('nueva fecha → reinicia',            () => expect(shouldReset('2026-06-05', '2026-06-06')).toBe(true));
  test('sin fecha guardada → reinicia',     () => expect(shouldReset(null, '2026-06-06')).toBe(true));
});
```

### 4. Persistencia en localStorage — Tests de integración

Verificar que la capa de Storage Repository lee y escribe correctamente. En browser, se puede usar el `localStorage` real; en Node.js (Jest), se mockea con `jest-localstorage-mock`.

```javascript
describe('StorageRepository', () => {
  beforeEach(() => localStorage.clear());

  test('getEntries devuelve array vacío cuando no hay datos', () => {
    expect(repo.getEntries()).toEqual([]);
  });
  test('saveEntries y getEntries son consistentes', () => {
    const entries = [{ id: 1, nombre: 'Almuerzo', calorias: 500 }];
    repo.saveEntries(entries);
    expect(repo.getEntries()).toEqual(entries);
  });
  test('getGoal devuelve null cuando no hay meta', () => {
    expect(repo.getGoal()).toBe(null);
  });
});
```

---

## Qué NO vale la pena testear

| Qué | Por qué no |
|-----|-----------|
| Que `textContent` escapa HTML | Es comportamiento del navegador, no del código de la app |
| Estilos CSS (colores de exceso) | Probar que un elemento tiene `color: red` es frágil y de poco valor; mejor un test de snapshot visual manual |
| Que `localStorage.setItem` funciona | Es comportamiento del navegador, no del código |
| Código boilerplate de inicialización | Sin lógica de negocio, sin valor en testear |

---

## Testing del reinicio diario a medianoche

El reinicio es el comportamiento más difícil de probar manualmente (¿vas a esperar hasta la medianoche?). La solución es **inyectar la fecha como dependencia**:

```javascript
// En lugar de:
function checkReset() {
  const today = new Date().toISOString().split('T')[0];  // ← difícil de mockear
  // ...
}

// Hazlo inyectable:
function checkReset(getTodayFn = () => new Date().toISOString().split('T')[0]) {
  const today = getTodayFn();
  // ...
}

// En el test:
checkReset(() => '2026-06-07');  // simula medianoche sin esperar
```

---

## Testing de rendimiento

No aplica para este sistema. Con ~50 entradas y `localStorage`, no hay carga que pueda degradar la experiencia. Si en el futuro se añade backend, un test de carga mínimo sería: 100 usuarios concurrentes registrando una entrada simultáneamente.

---

## Estrategia de datos de prueba

- **Unit tests**: datos inline hardcodeados en cada test — simple y explícito
- **Tests de integración**: `localStorage.clear()` en `beforeEach` garantiza aislamiento
- **E2E**: usar el flujo real de la usuaria (registrar 3 comidas, verificar total, eliminar una, verificar actualización)
- **No usar datos reales de producción** en tests — aquí no aplica porque no hay producción con datos de usuario reales

---

## Herramientas recomendadas

| Propósito | Herramienta | Alternativa |
|-----------|------------|------------|
| Unit/Integration tests | **Vitest** (si se usa Vite) o **Jest** | — |
| Mock de localStorage | `jest-localstorage-mock` o implementación propia con `Map` | — |
| E2E tests | **Playwright** (recomendado) | Cypress |
| Coverage | Integrado en Vitest/Jest | — |
