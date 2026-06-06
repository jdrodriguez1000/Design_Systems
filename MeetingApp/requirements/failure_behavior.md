# Failure Behavior — 010 Discovery
Fecha: 2026-06-01
Destino: input directo para Error & Exception Policy del 020 Specification Harness

## Escenarios de Fallo por Actor

### Empleado general (A-01)

| ID    | Escenario | Causa probable | Comportamiento esperado | Prioridad |
|-------|-----------|---------------|------------------------|-----------|
| SF-01 | Reserva activa cancelada por desplazamiento de prioridad | Una reserva de mayor prioridad (cliente externo, directorio o gerente con marca manual) reclama la misma sala en el mismo horario | El sistema cancela la reserva del empleado y le envía notificación in-app con razón explícita ("reasignada por reunión prioritaria") y oferta de sala alternativa disponible en el mismo horario, si existe. Si no hay alternativa disponible, la notificación se envía igualmente sin oferta de alternativa. | alta |
| SF-02 | No hay salas disponibles para el horario solicitado | Todas las salas están ocupadas en el intervalo solicitado (ya sea por reservas activas o por solapamiento de prioridades) | El sistema muestra mensaje de no disponibilidad para ese horario y sugiere los próximos horarios con al menos una sala libre. | media |

### Administradora (A-02)

| ID    | Escenario | Causa probable | Comportamiento esperado | Prioridad |
|-------|-----------|---------------|------------------------|-----------|
| SF-03 | Caída total del sistema | Falla de infraestructura, error crítico de aplicación u otro evento que deja el sistema inaccesible | El sistema no está disponible para reservas. Los usuarios deben usar el fallback: contactar a Ana por WhatsApp para coordinar salas. Cuando el sistema se restablece, Ana recibe notificación in-app informando la recuperación. [Causa marcada como inferencia: tipo de fallo no especificado en transcript; el mecanismo de detección de restablecimiento queda pendiente de definición en el 020.] | alta |
| SF-04 | Registro de reservas manuales tomadas durante la caída | La administradora necesita ingresar al sistema las reservas coordinadas por WhatsApp durante la caída, con las horas originales (pasadas) | El sistema permite a la administradora crear reservas con hora de inicio en el pasado. Antes de confirmar, el sistema muestra alerta de conflicto si ya existe otra reserva para la misma sala y horario (solapamiento parcial o total). La administradora decide si confirmar o no. | alta |

### Gerente general (A-03)

| ID    | Escenario | Causa probable | Comportamiento esperado | Prioridad |
|-------|-----------|---------------|------------------------|-----------|
| SF-05 | Reserva prioritaria del gerente desplaza una reserva activa existente | El gerente crea una reserva prioritaria (o el sistema la clasifica automáticamente como tal) para una sala ya ocupada por una reserva de menor prioridad | El sistema cancela la reserva de menor prioridad y notifica in-app a la persona desplazada con razón explícita y oferta de sala alternativa disponible en el mismo horario. Si no existe alternativa, la notificación se envía igualmente sin oferta. | alta |
| SF-06 | Dos reservas de igual prioridad máxima compiten por la misma sala | Dos usuarios con reservas de nivel prioritario (ambas de tipo cliente externo / directorio, o ambas marcadas por el gerente) intentan ocupar la misma sala en el mismo horario | La reserva creada primero (mayor antigüedad de timestamp de creación) mantiene la sala. La segunda reserva es rechazada o desplazada. El sistema resuelve esto de forma autónoma; la administradora no interviene. [Comportamiento exacto del rechazo de la segunda reserva — mensaje al solicitante — queda pendiente de especificación en el 020.] | alta |

## Escenarios Globales (no atribuibles a un actor específico)

| ID    | Escenario | Comportamiento esperado | Prioridad |
|-------|-----------|------------------------|-----------|
| SG-01 | Intento de reserva que viola la regla de exclusividad por usuario (solapamiento) | El sistema detecta que el usuario ya tiene una reserva activa que se solapa en horario con la nueva solicitud. Bloquea la nueva reserva e informa al usuario del conflicto. | alta |
| SG-02 | Restablecimiento del sistema tras caída | Una vez que el sistema vuelve a estar operativo, notifica in-app a la administradora (A-02). El resto de usuarios retoma el uso normal del sistema. [Mecanismo de detección del restablecimiento no definido en el transcript — pendiente de especificación en el 020.] | alta |

## Items sin Respuesta del Cliente

| Escenario | Pregunta pendiente | Impacto en el 020 |
|-----------|-------------------|--------------------|
| SF-03 / SG-02: Restablecimiento del sistema | ¿Cómo detecta el sistema que se ha restablecido? ¿Proceso automático de health-check, arranque manual, o evento externo? | El 020 debe definir el mecanismo técnico antes de especificar la notificación de recuperación a A-02. |
| SF-06: Empate de prioridad máxima — segunda reserva rechazada | ¿Qué mensaje recibe el solicitante de la segunda reserva cuando pierde el empate? ¿Se le ofrece alternativa automáticamente, o solo se le informa del rechazo? | El 020 debe especificar el comportamiento de rechazo para el caso de empate, distinguiéndolo del desplazamiento estándar. |
| SF-04: Reserva retroactiva — conflicto detectado | Cuando la administradora ve la alerta de conflicto al registrar una reserva retroactiva, ¿puede confirmarla de todas formas (forzando el registro) o el sistema bloquea definitivamente? | El 020 debe definir si la alerta es informativa (la administradora decide) o bloqueante (el sistema impide la confirmación). |
