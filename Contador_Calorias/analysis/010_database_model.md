# Modelo de Base de Datos — Contador de Calorías

## Tipo de almacenamiento recomendado: localStorage (clave-valor en el navegador)

Este sistema **no necesita una base de datos tradicional**. Su "base de datos" es el `localStorage` del navegador — un almacén clave-valor que persiste entre sesiones sin necesidad de servidor.

---

## Por qué localStorage y no una base de datos real

El patrón de acceso a datos de este sistema es extremadamente simple:

- **Escritura**: agregar una entrada al historial del día, guardar/actualizar la meta calórica
- **Lectura**: leer el historial completo del día, leer la meta calórica
- **Eliminación**: borrar una entrada puntual del historial, borrar todo el historial (reinicio diario)

No hay queries complejas, joins, búsquedas por texto, ni agregaciones más allá de una suma. `localStorage` maneja esto con naturalidad absoluta.

### ¿Por qué no IndexedDB?
IndexedDB sería la elección correcta si los datos crecieran mucho (miles de entradas) o si se necesitara búsqueda compleja. Para un historial diario de ~10-20 entradas que se borra cada noche, IndexedDB añade complejidad de API sin aportar nada.

### ¿Por qué no SQLite en el servidor?
Sin backend no hay servidor. Y sin servidor no hay SQLite. La arquitectura SPA frontend-only excluye esta opción por diseño.

---

## Esquema de datos en localStorage

### Clave 1: `calorie_tracker_entries`
**Tipo**: Array de objetos JSON serializado como string

```json
[
  {
    "id": "uuid-o-timestamp",
    "nombre": "Desayuno: avena con fruta",
    "calorias": 320,
    "timestamp": "2026-06-06T08:30:00"
  },
  {
    "id": "uuid-o-timestamp-2",
    "nombre": "Snack: manzana",
    "calorias": 80,
    "timestamp": "2026-06-06T10:15:00"
  }
]
```

### Clave 2: `calorie_tracker_goal`
**Tipo**: Número entero serializado como string (o null si no está configurado)

```
"1800"
```

### Clave 3: `calorie_tracker_date`
**Tipo**: String con fecha en formato ISO (YYYY-MM-DD)

```
"2026-06-06"
```

Esta clave es crítica para el **reinicio diario**: al cargar la app, se compara la fecha guardada con `new Date()`. Si son días diferentes, se borran las entradas y se actualiza la fecha.

---

## Reglas de negocio implementadas en la capa de datos

| Regla | Implementación |
|-------|---------------|
| RN-14: Historial persiste ante cierre | `localStorage` persiste por diseño entre sesiones del navegador |
| RN-16: Historial se borra en reinicio diario | Borrar `calorie_tracker_entries` y actualizar `calorie_tracker_date` |
| RN-19: Meta calórica persiste tras reinicio | No se toca `calorie_tracker_goal` durante el reinicio |
| RN-20: Zona horaria local para medianoche | `new Date()` en JavaScript usa la zona horaria del sistema operativo |

---

## Estrategia de indexación

`localStorage` no tiene índices — es acceso directo por clave. Esto no es un problema porque:
1. El volumen de datos es trivial (máximo ~50 entradas al día para un uso intensivo)
2. La operación más costosa (sumar calorías) es O(n) sobre un array pequeño: instantánea

---

## Evolución futura del modelo

Si en el futuro se implementa DF-04 (sincronización entre dispositivos), el modelo migra a:

```
localStorage (caché local)  ←→  API Backend  ←→  PostgreSQL / Firestore
```

El campo `id` en cada entrada (UUID o timestamp) ya está pensado para esta migración: cuando haya backend, ese `id` se convierte en la clave primaria en la base de datos del servidor.

---

## Conceptos aplicados

**BASE vs ACID**: `localStorage` no ofrece transacciones ACID. Si la escritura se interrumpe a mitad (improbable en este contexto), podría haber datos parciales. Para este dominio (calorías personales diarias) esto es aceptable — no es un sistema bancario. El modelo es **BASE**: básicamente disponible, estado eventual consistente.

**Serialización**: `localStorage` solo guarda strings. La serialización/deserialización con `JSON.stringify`/`JSON.parse` es la "ORM" de este sistema.
