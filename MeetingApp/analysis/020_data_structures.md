# Estructuras de Datos — MeetingApp

## El problema central: detección de solapamiento de intervalos

A diferencia de Contador_Calorias (donde las estructuras eran triviales), este sistema tiene un problema algorítmico real: **detectar eficientemente si dos intervalos de tiempo se solapan**.

---

## 1. Intervalo temporal (par de timestamps)

**¿Dónde se usa?**: cada reserva tiene `[fecha_inicio, fecha_fin)`. La operación crítica es: ¿se solapa el intervalo de una nueva reserva con alguna reserva activa existente?

**La fórmula de solapamiento** (dos intervalos [A_inicio, A_fin) y [B_inicio, B_fin) se solapan si y solo si):

```
A_inicio < B_fin  Y  B_inicio < A_fin
```

O equivalentemente (los intervalos NO se solapan si):

```
A_fin <= B_inicio  O  B_fin <= A_inicio
```

Esta es la condición que se usa en la cláusula WHERE de la query de detección. La implementación en SQL:

```sql
WHERE NOT (fecha_fin <= $inicio OR fecha_inicio >= $fin)
-- equivalente a:
WHERE fecha_inicio < $fin AND fecha_fin > $inicio
```

**Complejidad**: O(n) con escaneo lineal de todas las reservas activas de una sala. Con el índice en `(sala_id, estado, fecha_inicio, fecha_fin)`, PostgreSQL puede usar un index scan y acercarse a O(log n + k) donde k es el número de solapamientos.

---

## 2. Array ordenado / Sorted List (conceptual en la lógica de negocio)

**¿Dónde se usa?**: cuando se necesita sugerir "los próximos horarios disponibles" (OV-06, ACP-06), el algoritmo debe encontrar los huecos libres en el calendario de una sala.

**Algoritmo para encontrar próximos slots disponibles**:

```python
def find_next_available_slots(sala_id, from_datetime, duration_minutes, max_results=3):
    # 1. Obtener reservas activas futuras de la sala, ordenadas por inicio
    reservas = repo.find_active_from(sala_id, from_datetime)  # O(n log n) — ya ordenadas en DB

    # 2. Recorrer los huecos entre reservas consecutivas
    slots = []
    current = from_datetime
    for reserva in reservas:
        gap = (reserva.fecha_inicio - current).total_minutes()
        if gap >= duration_minutes:
            slots.append(current)
            if len(slots) >= max_results:
                break
        current = max(current, reserva.fecha_fin)

    return slots
```

Complejidad: O(n) donde n es el número de reservas futuras — lineal en el peor caso, pero n es pequeño (25 usuarios, 3 salas).

---

## 3. Tabla hash / Dictionary (resolución de prioridades)

**¿Dónde se usa?**: el mapa de permisos por rol y la clasificación automática de tipo_reunion → tipo_prioridad son tablas hash estáticas.

```python
PRIORIDAD_POR_TIPO_REUNION = {
    'cliente_externo': 'prioritaria',
    'directorio':      'prioritaria',
    'interna':         'estandar',
    None:              'estandar'
}

# Lookup O(1) en lugar de if/else if/else
def assign_priority(tipo_reunion: str, marcada_urgente: bool) -> str:
    if marcada_urgente:
        return 'prioritaria'
    return PRIORIDAD_POR_TIPO_REUNION.get(tipo_reunion, 'estandar')
```

**Complejidad**: O(1) para lookup — constante independientemente del número de tipos de reunión.

---

## 4. Queue (Cola) para notificaciones

**¿Dónde se usa?**: las notificaciones in-app no deben bloquear la transacción de reserva. La notificación se encola y se procesa después.

**En la versión más simple** (adecuada para este tamaño): la tabla `notificaciones_inapp` actúa como una cola persistente. El cliente las lee con polling al cargar la app.

```python
# Encolar notificación dentro de la transacción de reserva
# (simple — sin message broker externo para este tamaño)
def enqueue_notification(destinatario_id, tipo, mensaje, sala_alt=None):
    db.execute(
        "INSERT INTO notificaciones_inapp (destinatario_id, tipo, mensaje, sala_alternativa_id) VALUES (?,?,?,?)",
        [destinatario_id, tipo, mensaje, sala_alt]
    )
    # Esto ocurre dentro de BEGIN...COMMIT de la reserva
    # Si la transacción falla, la notificación también se revierte — correcto
```

Si en el futuro las notificaciones deben ser en tiempo real (DF-02 con push externo), esta cola evoluciona a Redis Queue o RabbitMQ.

---

## 5. UUID como identificador de entidades

**¿Dónde se usa?**: todas las entidades usan UUID v4 como clave primaria.

**¿Por qué UUID y no autoincrement integer?**

A diferencia de Contador_Calorias (un sistema de usuario único donde `Date.now()` era suficiente), aquí hay múltiples actores concurrentes. El autoincrement en PostgreSQL funciona con secuencias — seguro, pero requiere un roundtrip a la DB para obtener el ID. UUID v4 se puede generar en la aplicación antes de insertar, permitiendo incluir el ID en el evento de dominio antes de hacer el INSERT.

Además, si en el futuro se añaden más servidores o una migración de datos, los UUIDs garantizan unicidad global sin coordinación.

---

## 6. Conjunto (Set) para verificación de roles y permisos

**¿Dónde se usa?**: el sistema RBAC usa sets para representar los permisos de cada rol.

```python
# Set permite O(1) para "¿tiene este permiso?"
permisos_gerente = {'reservar', 'consultar', 'cancelar_propia', 'marcar_urgente'}

# O(1) — más eficiente que lista con O(n)
tiene_permiso = 'marcar_urgente' in permisos_gerente  # True
```

---

## Estructuras que se consideraron y descartaron

| Estructura | Por qué no aplica |
|-----------|-------------------|
| **Árbol de intervalos (Interval Tree)** | Permite O(log n + k) para búsqueda de solapamientos. Válido si hubiera miles de reservas. Para 25 usuarios + índice de BD, el overhead de implementar un interval tree no vale la pena |
| **Grafo** | Útil para modelar relaciones complejas entre entidades. Las relaciones aquí son simples (una reserva tiene un usuario y una sala) — no hay grafos |
| **Bloom Filter** | Verificación probabilística de pertenencia. No aplica: las queries de disponibilidad necesitan respuestas exactas, no probabilísticas |
