# Modelo de Base de Datos — MeetingApp

## Tipo recomendado: PostgreSQL (Base de datos relacional)

La elección es clara e inapelable para este sistema: **base de datos relacional con soporte ACID completo**.

---

## Por qué relacional y no NoSQL

El argumento central es la **lógica de conflictos**. Para determinar si una sala está disponible en un horario dado, la query es:

```sql
SELECT COUNT(*) FROM reservas
WHERE sala_id = $1
  AND estado = 'activa'
  AND NOT (fecha_fin <= $2 OR fecha_inicio >= $3);
```

Esta es una **query de solapamiento de intervalos** — un problema perfectamente modelado en SQL. En una base de datos de documentos (MongoDB), este tipo de query requiere leer muchos documentos y filtrar en la aplicación, o estructuras de datos ad-hoc que replican lo que SQL hace nativamente.

**El segundo argumento es la atomicidad**: cuando una reserva prioritaria desplaza a una estándar, ocurren 3-4 operaciones en la misma transacción. Si falla cualquiera, todo se revierte. Esto es ACID. No hay equivalente simple en MongoDB o DynamoDB sin complejidad adicional.

*Como se aprendió en Contador_Calorias con las entidades calculadas: `disponible_en_horario` es un valor calculado, no almacenado. Aquí aplica exactamente lo mismo: la disponibilidad de una sala se calcula al momento de la consulta, no se almacena como un campo en la tabla de salas.*

---

## Esquema principal

```sql
-- Tabla: usuarios
CREATE TABLE usuarios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre_completo TEXT NOT NULL,
    nombre_usuario  TEXT NOT NULL UNIQUE,
    password_hash   TEXT NOT NULL,
    rol             TEXT NOT NULL CHECK (rol IN ('empleado_general', 'gerente', 'administradora')),
    activo          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Tabla: salas (datos fijos — 3 salas)
CREATE TABLE salas (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre       TEXT NOT NULL UNIQUE,
    capacidad    INTEGER NOT NULL CHECK (capacidad > 0),
    equipamiento TEXT[] NOT NULL DEFAULT '{}'  -- ['proyector', 'videoconferencia', 'pizarra']
);

-- Tabla: reservas (entidad central)
CREATE TABLE reservas (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    usuario_id           UUID NOT NULL REFERENCES usuarios(id),
    sala_id              UUID NOT NULL REFERENCES salas(id),
    fecha_inicio         TIMESTAMPTZ NOT NULL,
    fecha_fin            TIMESTAMPTZ NOT NULL CHECK (fecha_fin > fecha_inicio),
    tipo_prioridad       TEXT NOT NULL CHECK (tipo_prioridad IN ('estandar', 'prioritaria')),
    tipo_reunion         TEXT CHECK (tipo_reunion IN ('interna', 'cliente_externo', 'directorio')),
    estado               TEXT NOT NULL CHECK (estado IN ('activa', 'cancelada', 'retroactiva')),
    es_retroactiva       BOOLEAN NOT NULL DEFAULT FALSE,
    timestamp_creacion   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT horario_valido CHECK (
        EXTRACT(HOUR FROM fecha_inicio) >= 8 AND
        EXTRACT(HOUR FROM fecha_fin) <= 19
    )
);

-- Tabla: desplazamientos (registro de causalidad)
CREATE TABLE desplazamientos (
    id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reserva_cancelada_id   UUID NOT NULL REFERENCES reservas(id),
    reserva_entrante_id    UUID NOT NULL REFERENCES reservas(id),
    motivo                 TEXT NOT NULL,
    timestamp_desplazamiento TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Tabla: notificaciones_inapp
CREATE TABLE notificaciones_inapp (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    destinatario_id  UUID NOT NULL REFERENCES usuarios(id),
    tipo             TEXT NOT NULL,
    mensaje          TEXT NOT NULL,
    sala_alternativa_id UUID REFERENCES salas(id),
    leida            BOOLEAN NOT NULL DEFAULT FALSE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Estrategia de indexación

```sql
-- Índice principal para detección de solapamientos en una sala
-- Query crítica: "¿hay reservas activas en esta sala en este intervalo?"
CREATE INDEX idx_reservas_sala_estado_horario
ON reservas(sala_id, estado, fecha_inicio, fecha_fin);

-- Índice para la regla de exclusividad por usuario
-- Query: "¿tiene este usuario reservas activas en este intervalo?"
CREATE INDEX idx_reservas_usuario_estado
ON reservas(usuario_id, estado);

-- Índice para notificaciones no leídas de un usuario
CREATE INDEX idx_notificaciones_destinatario_leida
ON notificaciones_inapp(destinatario_id, leida)
WHERE leida = FALSE;

-- Índice para timestamp_creacion (desempate de prioridad)
CREATE INDEX idx_reservas_timestamp_creacion
ON reservas(timestamp_creacion);
```

**¿Por qué índice en `(sala_id, estado, fecha_inicio, fecha_fin)`?**

La query de detección de solapamiento se ejecuta en cada intento de reserva. Sin índice, PostgreSQL haría un seq scan de toda la tabla de reservas. Con 25 usuarios haciendo ~5 reservas al día, la tabla crecerá a ~125 filas/día, ~46,000/año. El índice convierte una búsqueda O(n) en O(log n) — innecesario en el corto plazo pero buena práctica.

---

## Locking pesimista para concurrencia

Este es el aspecto más crítico del modelo de datos. Si dos usuarios intentan reservar la misma sala al mismo tiempo, puede ocurrir una **race condition** donde ambas transacciones leen "disponible" antes de que ninguna escriba.

**Solución**: `SELECT ... FOR UPDATE` en la sala específica antes de verificar solapamiento:

```sql
BEGIN;
-- Bloquear la sala para evitar race conditions
SELECT id FROM salas WHERE id = $sala_id FOR UPDATE;

-- Verificar solapamiento
SELECT COUNT(*) FROM reservas
WHERE sala_id = $sala_id
  AND estado = 'activa'
  AND NOT (fecha_fin <= $inicio OR fecha_inicio >= $fin);

-- Si no hay conflicto, insertar
INSERT INTO reservas (...) VALUES (...);
COMMIT;
```

Mientras esta transacción tiene el lock, ninguna otra transacción puede modificar la misma sala. La segunda transacción espera y luego ve la reserva ya creada — detecta el conflicto y rechaza correctamente.

---

## Alternativas consideradas

| Base de datos | ¿Sirve? | Por qué no es la primera opción |
|--------------|---------|----------------------------------|
| **MySQL** | Sí | PostgreSQL tiene mejor soporte para arrays (equipamiento), `TIMESTAMPTZ`, y advisory locks |
| **MongoDB** | Parcialmente | Transacciones multi-documento disponibles desde v4.0, pero la query de solapamiento de intervalos es más natural en SQL |
| **SQLite** | Solo para desarrollo | Sin concurrencia real (writer lock global); inapropiado para un servidor multiusuario |
| **Redis** | Como caché complementaria | Útil para cachear disponibilidad si el volumen crece, pero no como BD principal |

---

## Conceptos aplicados

**ACID completo**: cada operación de negocio compleja (reserva + posible desplazamiento + notificación) ocurre dentro de una transacción. Si falla en el paso 2 de 3, todo se revierte al estado inicial.

**Normalización**: el esquema está en 3FN. Los datos de sala no se repiten en cada reserva — se referencian por `sala_id`. Esto evita inconsistencias si el nombre de una sala cambia.

**Desnormalización controlada**: el campo `tipo_prioridad` en la tabla `reservas` es una desnormalización deliberada — podría derivarse de `tipo_reunion` + `rol del usuario`, pero almacenarlo explícitamente simplifica las queries de desplazamiento y el desempate por timestamp.
