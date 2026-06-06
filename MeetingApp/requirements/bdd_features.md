# BDD Feature Files — 020 Specification
Fecha: 2026-06-01
Estado: DRAFT
Basado en: specification/spec_analysis_report.md

## Resumen de cobertura

| Actor (ID) | Escenarios camino feliz | Escenarios caso de borde |
|------------|------------------------|--------------------------|
| [AC-01] Empleado general | 3 | 3 |
| [AC-02] Administradora | 4 | 3 |
| [AC-03] Gerente general | 3 | 3 |
| **Total** | **10** | **9** |

---

## Feature: Empleado general (AC-01)

### Escenarios de Camino Feliz

#### SC-01 — Consulta de disponibilidad de salas
**Fuente:** CF-01
**Actor:** AC-01 — Empleado general

```gherkin
Dado que el empleado general está autenticado en el sistema
  Y quiere verificar disponibilidad de salas para un horario dado
Cuando consulta las salas disponibles indicando fecha, hora de inicio y hora de fin
Entonces el sistema muestra las 3 salas con nombre, capacidad y equipamiento
  Y para cada sala indica si está disponible o reservada en ese intervalo
```

#### SC-02 — Reserva exitosa de sala disponible
**Fuente:** CF-02
**Actor:** AC-01 — Empleado general

```gherkin
Dado que el empleado general está autenticado en el sistema
  Y existe al menos una sala disponible para el horario solicitado
  Y el empleado general no tiene otra reserva activa que se solape con ese horario
Cuando reserva la sala seleccionada para ese horario
Entonces el sistema crea la reserva activa a nombre del empleado general
  Y el empleado general recibe notificación in-app de confirmación inmediata
```

#### SC-03 — Reserva con poca anticipación (sin tiempo mínimo)
**Fuente:** CF-03
**Actor:** AC-01 — Empleado general

```gherkin
Dado que el empleado general está autenticado en el sistema
  Y existe una sala disponible para un horario que comienza en menos de 2 horas
Cuando reserva esa sala para el horario próximo
Entonces el sistema acepta la reserva sin requerir anticipación mínima
  Y el empleado general recibe notificación in-app de confirmación inmediata
```

### Escenarios de Caso de Borde

#### SE-01 — Reserva activa cancelada por desplazamiento de prioridad
**Fuente:** CB-01
**Actor:** AC-01 — Empleado general

```gherkin
Dado que el empleado general tiene una reserva activa para una sala en un horario determinado
  Y una reserva prioritaria reclama esa misma sala en ese mismo horario
Cuando el sistema ejecuta el desplazamiento automático
Entonces el sistema cancela la reserva activa del empleado general
  Y el empleado general recibe notificación in-app con la razón explícita del desplazamiento
  Y si existe una sala alternativa disponible en el mismo horario, la notificación la incluye
  Y si no existe sala alternativa, la notificación se envía igualmente sin oferta de alternativa
```

#### SE-02 — Sin disponibilidad para el horario solicitado
**Fuente:** CB-02
**Actor:** AC-01 — Empleado general

```gherkin
Dado que el empleado general está autenticado en el sistema
  Y todas las salas están reservadas para el horario solicitado
Cuando intenta reservar una sala para ese horario
Entonces el sistema muestra mensaje de no disponibilidad para ese horario
  Y el sistema sugiere los próximos horarios con al menos una sala libre
```

#### SE-03 — Intento de reserva con solapamiento horario del mismo usuario
**Fuente:** CB-03
**Actor:** AC-01 — Empleado general

```gherkin
Dado que el empleado general tiene una reserva activa en un horario determinado
Cuando intenta crear una nueva reserva en un horario que se solapa con esa reserva activa
Entonces el sistema bloquea la nueva reserva
  Y informa al empleado general del conflicto de solapamiento con su reserva activa existente
```

---

## Feature: Administradora (AC-02)

### Escenarios de Camino Feliz

#### SC-04 — Creación de cuenta de usuario
**Fuente:** CF-04
**Actor:** AC-02 — Administradora

```gherkin
Dado que la administradora está autenticada en el sistema
  Y necesita registrar un nuevo miembro de la empresa
Cuando crea una cuenta de usuario indicando nombre, nombre de usuario, contraseña y rol
Entonces el sistema registra el nuevo usuario con el rol indicado
  Y la administradora recibe confirmación de creación exitosa
```

#### SC-05 — Notificación de restablecimiento del sistema
**Fuente:** CF-05
**Actor:** AC-02 — Administradora

```gherkin
Dado que el sistema ha estado caído y se restablece
Cuando el sistema completa su arranque y el health-check automático detecta operatividad
Entonces el sistema envía notificación in-app a la administradora informando la recuperación del sistema
```

#### SC-06 — Registro de reserva retroactiva sin conflicto
**Fuente:** CF-06
**Actor:** AC-02 — Administradora

```gherkin
Dado que la administradora está autenticada en el sistema
  Y necesita registrar una reserva coordinada por WhatsApp durante una caída
  Y no existe otra reserva para la misma sala y horario en el sistema
Cuando crea una reserva retroactiva indicando una hora de inicio en el pasado
Entonces el sistema crea la reserva retroactiva
  Y la administradora recibe confirmación de registro exitoso
```

#### SC-07 — Registro de reserva retroactiva con conflicto (alerta informativa)
**Fuente:** CF-07
**Actor:** AC-02 — Administradora

```gherkin
Dado que la administradora está autenticada en el sistema
  Y necesita registrar una reserva retroactiva para una sala y horario
  Y ya existe otra reserva para esa misma sala en ese horario en el sistema
Cuando intenta crear la reserva retroactiva
Entonces el sistema muestra alerta de conflicto con la reserva existente antes de confirmar
  Y la administradora puede confirmar la reserva retroactiva de todas formas
  Y el sistema registra la reserva retroactiva al confirmar
```

### Escenarios de Caso de Borde

#### SE-04 — Restablecimiento del sistema tras caída (health-check automático)
**Fuente:** CB-04
**Actor:** AC-02 — Administradora

```gherkin
Dado que el sistema acaba de completar su proceso de arranque tras una caída
Cuando el health-check automático detecta que el sistema está operativo
Entonces el sistema envía notificación in-app a la administradora informando el restablecimiento
  Y los demás usuarios pueden retomar el uso normal del sistema
```

#### SE-05 — Conflicto detectado al registrar reserva retroactiva
**Fuente:** CB-05
**Actor:** AC-02 — Administradora

```gherkin
Dado que la administradora intenta registrar una reserva retroactiva
  Y ya existe una reserva para la misma sala y horario en el sistema
Cuando el sistema detecta el solapamiento
Entonces el sistema muestra alerta informativa de conflicto antes de confirmar
  Y la administradora puede optar por confirmar la reserva retroactiva a pesar del conflicto
  Y la alerta no bloquea definitivamente la creación de la reserva retroactiva
```

#### SE-06 — Reserva retroactiva viola exclusividad del propio usuario
**Fuente:** CB-06
**Actor:** AC-02 — Administradora

```gherkin
Dado que la administradora tiene una reserva activa en un horario determinado
Cuando intenta crear una reserva retroactiva en un horario que se solapa con esa reserva activa suya
Entonces el sistema bloquea la nueva reserva
  Y informa a la administradora del conflicto de solapamiento con su propia reserva activa
```

---

## Feature: Gerente general (AC-03)

### Escenarios de Camino Feliz

#### SC-08 — Reserva con prioridad automática (cliente externo o directorio)
**Fuente:** CF-08
**Actor:** AC-03 — Gerente general

```gherkin
Dado que la gerente general está autenticada en el sistema
  Y quiere reservar una sala para una reunión con un cliente externo o con el directorio
  Y la sala está disponible para el horario solicitado
Cuando crea la reserva indicando el tipo de reunión (cliente externo o directorio)
Entonces el sistema clasifica automáticamente la reserva como prioritaria
  Y crea la reserva activa para la gerente general
  Y la gerente general recibe notificación in-app de confirmación inmediata
```

#### SC-09 — Marcado manual de reunión interna como urgente
**Fuente:** CF-09
**Actor:** AC-03 — Gerente general

```gherkin
Dado que la gerente general está autenticada en el sistema
  Y tiene una reserva activa para una reunión interna
Cuando marca manualmente esa reunión como urgente
Entonces el sistema eleva la prioridad de esa reserva a prioritaria
  Y solo el rol gerente general puede ejecutar esta acción
```

#### SC-10 — Desplazamiento automático de reserva estándar por reserva prioritaria
**Fuente:** CF-10
**Actor:** AC-03 — Gerente general

```gherkin
Dado que la gerente general está autenticada en el sistema
  Y una sala está ocupada por una reserva activa estándar en un horario determinado
  Y la gerente general crea una reserva prioritaria para esa misma sala y horario
Cuando el sistema procesa la reserva prioritaria
Entonces el sistema cancela automáticamente la reserva estándar (desplazamiento)
  Y crea la reserva activa prioritaria de la gerente general
  Y el usuario desplazado recibe notificación in-app con razón explícita del desplazamiento
  Y si existe sala alternativa disponible en el mismo horario, la notificación la incluye
  Y si no existe sala alternativa, la notificación se envía igualmente sin oferta de alternativa
```

### Escenarios de Caso de Borde

#### SE-07 — Reserva prioritaria del gerente desplaza reserva activa
**Fuente:** CB-07
**Actor:** AC-03 — Gerente general

```gherkin
Dado que existe una reserva activa estándar para una sala en un horario determinado
Cuando la gerente general crea o tiene una reserva prioritaria para esa misma sala y horario
Entonces el sistema cancela la reserva de menor prioridad de forma automática
  Y el usuario desplazado recibe notificación in-app con razón explícita
  Y si existe sala alternativa disponible en el mismo horario, la notificación la incluye
  Y si no existe sala alternativa, la notificación se envía sin oferta de alternativa
```

#### SE-08 — Empate de prioridad máxima: segunda reserva rechazada
**Fuente:** CB-08
**Actor:** AC-03 — Gerente general

```gherkin
Dado que existe una reserva activa prioritaria para una sala en un horario determinado
Cuando un segundo usuario crea una reserva también prioritaria para esa misma sala y horario
Entonces el sistema mantiene la reserva creada primero (mayor antigüedad de timestamp de creación)
  Y rechaza la segunda reserva de forma automática sin intervención de la administradora
  Y el solicitante de la segunda reserva recibe notificación in-app con el rechazo
  Y si existe sala alternativa disponible en el mismo horario, la notificación la incluye
  Y si no existe sala alternativa, la notificación se envía sin oferta de alternativa
```

#### SE-09 — Gerente intenta reserva con solapamiento horario propio
**Fuente:** CB-09
**Actor:** AC-03 — Gerente general

```gherkin
Dado que la gerente general tiene una reserva activa en un horario determinado
Cuando intenta crear una nueva reserva en un horario que se solapa con esa reserva activa
Entonces el sistema bloquea la nueva reserva
  Y informa a la gerente general del conflicto de solapamiento con su reserva activa existente
  Y la regla de exclusividad aplica de forma uniforme a todos los roles incluido el gerente
```
