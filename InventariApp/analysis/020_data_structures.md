# Estructuras de Datos — InventariApp

## El problema central: stock como agregado de un log inmutable

El modelo de datos de InventariApp es fundamentalmente un **ledger** (libro de contabilidad). El stock actual no es un número que se guarda — es el resultado de sumar y restar todos los movimientos del historial. Esto convierte el cálculo de stock en un problema de agregación sobre una lista ordenada.

---

## 1. Log de movimientos (Lista append-only / Event Log)

**¿Dónde se usa?**: tabla `movimientos` — la estructura de datos más importante del sistema.

**Característica clave**: solo se puede insertar, nunca modificar ni eliminar. Cada movimiento tiene un `tipo_impacto` (SUMA/RESTA) y una `cantidad`, y el stock actual es el resultado de agregar toda la lista.

```
movimientos del producto X (ordenados por fecha_hora_evento):
[
  { tipo: 'entrada',         tipo_impacto: 'SUMA',  cantidad: 100 },  → stock: 100
  { tipo: 'salida',          tipo_impacto: 'RESTA', cantidad: 30  },  → stock: 70
  { tipo: 'salida',          tipo_impacto: 'RESTA', cantidad: 50  },  → stock: 20
  { tipo: 'contra-movimiento', tipo_impacto: 'SUMA', cantidad: 50  }  → stock: 70
  (el contra-movimiento revierte la segunda salida — error de registro)
]

stock_actual = SUM(CASE tipo_impacto='SUMA' THEN +cantidad ELSE -cantidad END) = 70
```

**Complejidad**:
- Inserción: O(1) — siempre al final del log
- Cálculo de stock: O(n) donde n = número de movimientos del producto
- Con índice en `(producto_id, tipo_impacto, cantidad)`: PostgreSQL puede calcular el agregado en O(log n + n) donde la parte log n es el índice scan

**Por qué esta estructura y no un saldo actualizable**: si el stock fuera un campo editable en `productos`, podría quedar desincronizado con el historial si una transacción falla a mitad. Con el log inmutable, el saldo *es* el historial — nunca hay desincronización.

---

## 2. Máquina de estados (State Machine) para OrdenReposicion

**¿Dónde se usa?**: ciclo de vida de las órdenes de reposición.

Las transiciones son lineales y no reversibles:

```
    BORRADOR ──────────── confirm() ──────────→ CONFIRMADA
       │                                             │
       │                                          receive()
       │                                             │
       └───────────────────────────────────────→ RECIBIDA
                                                 (estado final)
```

**Representación como grafo de adyacencia (tabla hash):**

```python
VALID_TRANSITIONS = {
    'borrador':   frozenset(['confirmada']),
    'confirmada': frozenset(['recibida']),
    'recibida':   frozenset()  # estado final, sin transiciones
}

# Lookup O(1): ¿es válida esta transición?
def is_valid(from_state: str, to_state: str) -> bool:
    return to_state in VALID_TRANSITIONS.get(from_state, frozenset())
```

**Complejidad**: lookup O(1) con tabla hash. Con solo 3 estados, no importa en términos de rendimiento, pero la estructura hace imposible que el código llegue a un estado no definido.

---

## 3. Conjunto (Set) para flags de negocio y validaciones

**¿Dónde se usa?**: verificaciones de estado en la lógica de validación.

Los tipos válidos de movimiento, subtipos, estados de orden y estados de alerta se representan como sets para verificaciones O(1):

```python
TIPOS_QUE_BLOQUEAN_ENTRADAS = frozenset(['discontinuado'])
TIPOS_DE_IMPACTO = frozenset(['SUMA', 'RESTA'])
SUBTIPOS_ENTRADA = frozenset(['compra', 'devolucion_cliente'])
SUBTIPOS_SALIDA  = frozenset(['despacho', 'perdida_dano'])

# Verificación O(1) en vez de if/elif encadenados
def puede_recibir_entrada(producto: Producto) -> bool:
    return producto.estado not in TIPOS_QUE_BLOQUEAN_ENTRADAS
```

---

## 4. Tabla hash para caché de KPIs ejecutivos

**¿Dónde se usa?**: panel de estado general de Luis — consulta frecuente que agrega múltiples productos.

El panel necesita calcular en tiempo real:
1. Número de alertas activas (COUNT de alertas_stock_bajo WHERE estado='activa')
2. Número de excepciones en el mes (COUNT de movimientos WHERE flag_excepcion=TRUE AND fecha_hora_evento >= inicio_mes)

Estas dos queries son simples y con índices responden en <10ms para la escala del sistema (18 usuarios, ~1,000 movimientos/mes). No se necesita caché en v1, pero si el panel se vuelve lento, una tabla hash en Redis con TTL de 30 segundos bastaría:

```python
# Estructura de caché (si fuera necesaria en v2)
kpi_cache = {
    'alertas_activas': {
        'valor': 3,
        'calculado_en': datetime(2026, 6, 6, 10, 30, 0)
    },
    'excepciones_mes': {
        'valor': 7,
        'calculado_en': datetime(2026, 6, 6, 10, 30, 0)
    }
}
# Invalidación: al insertar cualquier movimiento o resolver/crear alerta
```

---

## 5. Cola conceptual para generación de alertas

**¿Dónde se usa?**: cuando `AlertEvaluator` necesita recalcular el estado de múltiples productos simultáneamente (ej: al modificar el stock mínimo de un producto que ya tiene movimientos).

La evaluación de alertas puede modelarse como un procesamiento de una cola de `(producto_id, nuevo_stock_minimo)` pares a evaluar. En v1, dado que solo hay un producto a evaluar por vez, no se materializa como una cola real — es conceptual.

Si en el futuro se añade importación masiva de cambios de stock mínimo (capacidad diferida), esta cola se convertiría en una job queue real (Celery + Redis).

---

## Estructuras consideradas y descartadas

| Estructura | Por qué no aplica |
|-----------|-------------------|
| **Árbol de intervalos** | Útil para detectar solapamientos de rangos temporales (como en MeetingApp). InventariApp no necesita detectar si dos movimientos se solapan en el tiempo. |
| **Heap / Cola de prioridad** | Útil cuando se necesita procesar elementos en orden de prioridad. Las alertas no tienen prioridad entre sí — todas son igualmente urgentes. |
| **Bloom Filter** | Verificación probabilística de pertenencia. No aplica: las verificaciones de stock y alertas necesitan respuestas exactas. |
| **Trie** | Búsqueda eficiente por prefijos de texto. El catálogo de productos no tiene búsqueda por prefijo en v1. |
