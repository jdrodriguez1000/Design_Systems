# Estructuras de Datos — Contador de Calorías

## Resumen

Este sistema usa estructuras simples. La complejidad técnica es baja por diseño — el dominio lo justifica. El valor educativo está en entender **por qué** estas estructuras y no otras, y cuáles son los trade-offs de cada elección.

---

## 1. Array (Lista de entradas de comida)

**¿Dónde se usa?**: `appState.entries` — el historial del día.

**¿Por qué un Array y no un Set o un Map?**

| Estructura | ¿Sirve aquí? | Razón |
|-----------|-------------|-------|
| **Array** | ✅ Sí | Orden de inserción importa (Sofía ve sus comidas en orden cronológico). Permite duplicados (comer lo mismo dos veces es válido). Suma con `.reduce()` es directa. |
| **Set** | ❌ No | Un Set garantiza unicidad. Pero dos "Almuerzo: pollo" son entradas legítimamente diferentes. |
| **Map** | ❌ No | Un Map es eficiente para búsqueda por clave, pero aquí nunca se busca por nombre — se itera todo para sumar. |

**Operaciones críticas y su complejidad**:

| Operación | Complejidad | Frecuencia |
|-----------|-------------|-----------|
| Agregar entrada (`push`) | O(1) amortizado | Varias veces al día |
| Sumar calorías (`reduce`) | O(n) | Cada render |
| Eliminar por ID (`filter`) | O(n) | Ocasional |
| Renderizar lista completa | O(n) | Cada render |

Con n ≤ ~50 entradas diarias, O(n) es insignificante. No es un problema de escala que valga la pena optimizar.

---

## 2. Número entero (Total consumido, Meta calórica, Saldo, Exceso)

**¿Dónde se usan?**: Todos los valores calóricos del sistema.

**¿Por qué entero y no flotante?**

Las calorías se expresan en kcal como números enteros en el dominio de la nutrición. Sofía ingresa "320", no "320.5". Usar flotantes introduciría errores de precisión de punto flotante que no tienen sentido en el dominio:

```javascript
// Con flotantes — resultado contraintuitivo:
1800.0 - 1799.9 = 0.09999999999998863  // ¿"Te quedan 0.09 kcal"?

// Con enteros — resultado correcto:
Math.floor(1800) - Math.floor(1799) = 1  // "Te queda 1 kcal"
```

**Validación**: el campo de entrada solo acepta `parseInt(input, 10)` — cualquier decimal ingresado se trunca o rechaza.

---

## 3. String (Nombre de la entrada y claves de localStorage)

**¿Dónde se usa?**: 
- Nombre de cada entrada de comida (texto libre, sin restricción de formato)
- Claves de `localStorage`: `'calorie_tracker_entries'`, `'calorie_tracker_goal'`, `'calorie_tracker_date'`

**Consideración importante — XSS**:
El nombre de la entrada es texto libre ingresado por el usuario que se renderiza en el DOM. Si se usa `innerHTML` para renderizarlo, una entrada como `<script>alert('xss')</script>` podría ejecutarse. La solución: siempre usar `textContent` en lugar de `innerHTML` para contenido ingresado por el usuario.

---

## 4. Objeto plano (Entidad Entrada de Comida)

**¿Dónde se usa?**: Cada elemento del array de entradas.

```javascript
{
  id: 1717689600000,     // timestamp como identificador único simple
  nombre: "Almuerzo",   // string libre
  calorias: 650          // entero > 0
}
```

**¿Por qué timestamp como ID y no UUID?**

Un UUID (como `"f47ac10b-58cc-4372-a567-0e02b2c3d479"`) es el estándar para sistemas distribuidos donde múltiples fuentes generan IDs sin coordinación. Aquí hay un único usuario en un único dispositivo: `Date.now()` (timestamp en milisegundos) cumple el mismo propósito con cero dependencias externas.

**Riesgo conocido**: si Sofía agrega dos entradas en el mismo milisegundo, hay colisión de IDs. Probabilidad en uso real: prácticamente cero. Si se quisiera eliminar este riesgo con cero overhead: `Date.now() + Math.random()`.

---

## 5. String de fecha ISO (Control del reinicio diario)

**¿Dónde se usa?**: `localStorage.getItem('calorie_tracker_date')` — detecta el cambio de día.

```javascript
// Formato: "2026-06-06"
const today = new Date().toISOString().split('T')[0];
const storedDate = StorageRepository.getLastDate();

if (storedDate !== today) {
  // Es un nuevo día — ejecutar reinicio
  resetDailyData();
  StorageRepository.saveDate(today);
}
```

**¿Por qué ISO 8601 (YYYY-MM-DD) y no otro formato?**

El formato ISO 8601 ordena cronológicamente cuando se comparan strings alfanuméricamente. `"2026-06-06" < "2026-06-07"` es `true`. Formatos como `"06/06/2026"` no tienen esta propiedad.

---

## Estructuras que NO se necesitan aquí

| Estructura | Cuándo sería útil | Por qué no aplica |
|-----------|-------------------|-------------------|
| **Cola (Queue)** | Procesar tareas en orden FIFO (ej: sistema de notificaciones) | No hay procesamiento asincrónico de tareas |
| **Árbol** | Categorías jerárquicas, DOM traversal | No hay jerarquía en los datos |
| **Grafo** | Relaciones entre entidades complejas (ej: red social) | Solo hay una entidad principal |
| **Heap / Cola de prioridad** | Scheduling, algoritmos de optimización | No hay prioridades entre datos |
| **Bloom Filter** | Verificar pertenencia en grandes conjuntos con bajo uso de memoria | El conjunto de datos es trivialmente pequeño |
