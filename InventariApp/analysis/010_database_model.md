# Modelo de Base de Datos — InventariApp

## Tipo recomendado: PostgreSQL (relacional con ACID completo)

La elección es directa: el sistema es inherentemente relacional. El catálogo de productos se relaciona con movimientos, que se relacionan con alertas, que se relacionan con órdenes. El stock se calcula con un `SUM()` SQL sobre la tabla de movimientos. No hay argumentos serios para una base de datos de documentos aquí.

---

## El insight central: stock como valor calculado

*Como se aprendió en Contador_Calorias: los valores derivables de la fuente de verdad no deben almacenarse por separado. Aquí es una regla de negocio explícita (RN-01): el stock actual se calcula siempre a partir del historial de movimientos y nunca se edita directamente.*

La tabla `productos` **no tiene** columna `stock_actual`. El stock se obtiene en cada consulta como:

```sql
SELECT
    p.id,
    p.nombre,
    COALESCE(SUM(
        CASE WHEN m.tipo_impacto = 'SUMA' THEN m.cantidad
             ELSE -m.cantidad
        END
    ), 0) AS stock_actual
FROM productos p
LEFT JOIN movimientos m ON m.producto_id = p.id
GROUP BY p.id;
```

Ventaja: imposible que el stock y el historial queden desincronizados. No hay "campo a editar directamente" porque no existe. La consistencia está garantizada por diseño, no por disciplina.

Desventaja: si la tabla de movimientos crece a millones de filas, esta query puede ser lenta. Solución: índice en `(producto_id, tipo_impacto, cantidad)` y, si escala, una vista materializada o un campo de saldo actualizado transaccionalmente.

---

## Esquema principal

```sql
-- Tabla: categorias
CREATE TABLE categorias (
    id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre  TEXT NOT NULL UNIQUE
);

-- Tabla: proveedores
CREATE TABLE proveedores (
    id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre    TEXT NOT NULL,
    telefono  TEXT,
    email     TEXT
);

-- Tabla: productos
CREATE TABLE productos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo_interno  TEXT NOT NULL UNIQUE,   -- inmutable: se asigna al crear, no cambia
    nombre          TEXT NOT NULL,
    categoria_id    UUID NOT NULL REFERENCES categorias(id),
    unidad_medida   TEXT NOT NULL,
    stock_minimo    INTEGER NOT NULL DEFAULT 0 CHECK (stock_minimo >= 0),
    estado          TEXT NOT NULL DEFAULT 'activo' CHECK (estado IN ('activo', 'discontinuado')),
    proveedor_1_id  UUID REFERENCES proveedores(id),
    proveedor_2_id  UUID REFERENCES proveedores(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
    -- SIN columna stock_actual: se calcula desde movimientos
);

-- Tabla: movimientos (append-only — nunca UPDATE ni DELETE)
CREATE TABLE movimientos (
    id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    producto_id               UUID NOT NULL REFERENCES productos(id),
    tipo                      TEXT NOT NULL CHECK (tipo IN ('entrada', 'salida', 'contra-movimiento')),
    subtipo                   TEXT CHECK (subtipo IN ('compra', 'devolucion_cliente', 'despacho', 'perdida_dano')),
    tipo_impacto              TEXT NOT NULL CHECK (tipo_impacto IN ('SUMA', 'RESTA')),
    -- tipo_impacto determina el signo al calcular el stock:
    -- SUMA: el movimiento incrementa el stock (entradas, contra-movimientos de salidas)
    -- RESTA: el movimiento decrementa el stock (salidas, contra-movimientos de entradas)
    cantidad                  INTEGER NOT NULL CHECK (cantidad > 0),
    fecha_hora_evento         TIMESTAMPTZ NOT NULL,  -- editable por el usuario, no puede ser futura
    timestamp_creacion        TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- generado por servidor, no editable
    flag_auditoria_retroactiva BOOLEAN NOT NULL DEFAULT FALSE,    -- auto: diferencia > 1 hora
    referencia                TEXT,
    nota                      TEXT,  -- obligatoria para contra-movimientos y excepciones
    flag_excepcion            BOOLEAN NOT NULL DEFAULT FALSE,
    nota_excepcion            TEXT,  -- obligatoria cuando flag_excepcion = TRUE
    movimiento_origen_id      UUID REFERENCES movimientos(id),    -- solo para contra-movimientos
    CONSTRAINT nota_obligatoria_excepcion
        CHECK (flag_excepcion = FALSE OR nota_excepcion IS NOT NULL),
    CONSTRAINT nota_obligatoria_contra
        CHECK (tipo != 'contra-movimiento' OR nota IS NOT NULL)
);

-- Tabla: alertas_stock_bajo
CREATE TABLE alertas_stock_bajo (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    producto_id        UUID NOT NULL REFERENCES productos(id),
    estado             TEXT NOT NULL DEFAULT 'activa' CHECK (estado IN ('activa', 'resuelta')),
    stock_al_disparo   INTEGER NOT NULL,
    fecha_disparo      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    fecha_resolucion   TIMESTAMPTZ,
    CONSTRAINT una_alerta_activa_por_producto
        UNIQUE (producto_id, estado) DEFERRABLE INITIALLY DEFERRED
    -- Este UNIQUE parcial garantiza RN-13: máximo 1 alerta activa por producto
    -- (En PostgreSQL se puede lograr más elegantemente con un índice parcial)
);

-- Índice parcial más eficiente para RN-13
CREATE UNIQUE INDEX idx_una_alerta_activa_por_producto
ON alertas_stock_bajo(producto_id)
WHERE estado = 'activa';

-- Tabla: ordenes_reposicion
CREATE TABLE ordenes_reposicion (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    producto_id         UUID NOT NULL REFERENCES productos(id),
    proveedor_id        UUID NOT NULL REFERENCES proveedores(id),
    alerta_origen_id    UUID NOT NULL REFERENCES alertas_stock_bajo(id),
    cantidad_sugerida   INTEGER NOT NULL CHECK (cantidad_sugerida > 0),
    cantidad_ajustada   INTEGER NOT NULL CHECK (cantidad_ajustada > 0),
    estado              TEXT NOT NULL DEFAULT 'borrador' CHECK (estado IN ('borrador', 'confirmada', 'recibida')),
    fecha_creacion      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    movimiento_recepcion_id UUID REFERENCES movimientos(id)
    -- Al pasar a 'recibida', se llena este campo con el movimiento de entrada creado automáticamente
);
```

---

## Estrategia de indexación

```sql
-- Índice principal: cálculo de stock y consulta de historial por producto
-- La query más frecuente del sistema
CREATE INDEX idx_movimientos_producto_id
ON movimientos(producto_id);

-- Índice para movimientos recientes con flag retroactivo (condición de datos no conciliados)
-- Query: "¿hay movimientos retroactivos en las últimas 24 horas?"
CREATE INDEX idx_movimientos_auditoria_retroactiva
ON movimientos(timestamp_creacion, flag_auditoria_retroactiva)
WHERE flag_auditoria_retroactiva = TRUE;

-- Índice para historial de excepciones (vista de Diana)
CREATE INDEX idx_movimientos_excepciones
ON movimientos(producto_id, fecha_hora_evento)
WHERE flag_excepcion = TRUE;

-- Índice para alertas activas (pantalla principal de Andrés y KPI de Luis)
CREATE INDEX idx_alertas_activas
ON alertas_stock_bajo(producto_id)
WHERE estado = 'activa';

-- Índice para órdenes activas por producto (verificar RN-19)
CREATE INDEX idx_ordenes_activas_por_producto
ON ordenes_reposicion(producto_id, alerta_origen_id)
WHERE estado IN ('borrador', 'confirmada');

-- Índice para filtrado de panel por categoría (consulta de Luis)
CREATE INDEX idx_productos_categoria_estado
ON productos(categoria_id, estado);
```

---

## Por qué este diseño y no otros

**¿Por qué `tipo_impacto` (SUMA/RESTA) y no solo `tipo` (entrada/salida)?**
El campo `tipo_impacto` hace explícito el signo del movimiento en la agregación. Un contra-movimiento puede ser SUMA (si revierte una salida) o RESTA (si revierte una entrada). Sin este campo, el cálculo de stock requiere lógica de aplicación que sigue el árbol de dependencias de movimientos — complejo y propenso a errores. Con `tipo_impacto`, el stock es `SUM(CASE WHEN tipo_impacto='SUMA' THEN cantidad ELSE -cantidad END)` — una query simple.

**¿Por qué el índice parcial para alertas activas?**
La query más frecuente para la pantalla principal es "¿qué alertas están activas ahora?". Un índice parcial `WHERE estado = 'activa'` es mucho más pequeño que un índice completo — solo contiene las alertas que importan. Las alertas resueltas quedan fuera del índice sin afectar la lectura de activas.

**¿Por qué PostgreSQL y no SQLite?**
SQLite tiene un writer lock global — con múltiples usuarios simultáneos (Andrés, Diana, Luis accediendo al panel), se formaría una cola de espera innecesaria. PostgreSQL maneja la concurrencia a nivel de transacción correctamente.

---

## Alternativas consideradas

| Base de datos | ¿Sirve? | Por qué no es la primera opción |
|--------------|---------|----------------------------------|
| **MySQL** | Sí | PostgreSQL tiene mejor soporte para `CHECK` constraints, `TIMESTAMPTZ`, índices parciales y `gen_random_uuid()` nativo |
| **MongoDB** | Parcialmente | El modelo de movimientos como documentos funciona, pero el cálculo de stock requeriría un aggregation pipeline complejo donde SQL es más directo |
| **SQLite** | Solo para desarrollo | Writer lock global, inapropiado para múltiples usuarios |
| **Firebase Realtime DB** | No | Sin transacciones ACID para el trio movimiento → stock → alerta |

---

## Conceptos aplicados

**ACID completo**: el trio (insertar movimiento + calcular stock + generar/resolver alerta) ocurre en una sola transacción. Si falla en el paso 2 de 3, todo se revierte.

**Normalización**: el esquema está en 3FN. El nombre del proveedor no se copia en cada movimiento — se referencia por `proveedor_id`.

**Append-only / Ledger pattern**: la tabla `movimientos` nunca recibe UPDATE ni DELETE. La corrección de errores se hace con nuevos registros (contra-movimientos). Esto garantiza auditoría total por diseño, no por disciplina.

**Índice parcial**: PostgreSQL permite índices sobre subconjuntos de filas (`WHERE estado = 'activa'`). Esto mantiene el índice pequeño y la query de pantalla principal rápida incluso cuando la tabla de alertas tenga miles de alertas resueltas históricas.
