# Product Acceptance Criteria — 020 Specification
Fecha: 2026-06-01
Estado: DRAFT
Basado en: specification/spec_analysis_report.md + specification/bdd_features.md

## Criterios de Aceptación por Actor

### Actor AC-01 — Empleado general

| ID     | Criterio | Escenario BDD | Condición de cumplimiento | Condición de fallo |
|--------|----------|---------------|--------------------------|-------------------|
| ACP-01 | Al consultar disponibilidad, el sistema muestra las 3 salas con nombre, capacidad y equipamiento, indicando cuáles están libres para el horario solicitado | SC-01 | El usuario ve las 3 salas con sus atributos y el estado disponible/reservada para el intervalo indicado | El sistema muestra menos de 3 salas, omite atributos de alguna sala, o no indica el estado de disponibilidad |
| ACP-02 | Al reservar una sala disponible, el sistema crea la reserva activa y entrega confirmación inmediata in-app | SC-02 | La reserva queda registrada como activa y el empleado recibe notificación in-app de confirmación en el mismo momento de la operación | La reserva no se crea, no queda en estado activa, o no se genera notificación in-app de confirmación |
| ACP-03 | El sistema acepta reservas sin tiempo mínimo de anticipación, incluso para horarios que comienzan en menos de 2 horas | SC-03 | Una reserva para un horario inminente (ej: dentro de 1 hora) es aceptada y confirmada sin mensaje de restricción de anticipación | El sistema rechaza la reserva por insuficiente anticipación o muestra mensaje de tiempo mínimo |
| ACP-04 | Cuando una reserva activa del empleado es cancelada por desplazamiento, recibe notificación in-app con la razón explícita del desplazamiento | SE-01 | La notificación in-app incluye la razón del desplazamiento en lenguaje comprensible (ej: "tu reserva fue reasignada por una reunión prioritaria") | La notificación no se envía, o se envía sin razón explícita del desplazamiento |
| ACP-05 | Cuando existe sala alternativa disponible en el mismo horario del desplazamiento, la notificación in-app la incluye | SE-01 | Si hay sala disponible en el mismo horario, la notificación menciona la alternativa; si no hay, la notificación se envía igualmente sin mención de alternativa | La notificación incluye una sala alternativa que no está realmente disponible, o no incluye alternativa cuando sí existe una |
| ACP-06 | Cuando no hay salas disponibles para el horario solicitado, el sistema muestra mensaje de no disponibilidad y sugiere próximos horarios con disponibilidad | SE-02 | El mensaje indica explícitamente la no disponibilidad para el horario solicitado y muestra al menos un horario alternativo con sala libre | El sistema no muestra mensaje, o muestra solo la no disponibilidad sin sugerir alternativas |
| ACP-07 | El sistema impide que un empleado cree una reserva que se solape con otra reserva activa suya | SE-03 | El sistema bloquea la reserva e informa al empleado del conflicto de solapamiento | El sistema permite la creación de una segunda reserva que solapa con la primera |

---

### Actor AC-02 — Administradora

| ID     | Criterio | Escenario BDD | Condición de cumplimiento | Condición de fallo |
|--------|----------|---------------|--------------------------|-------------------|
| ACP-08 | La administradora puede crear cuentas de usuario con nombre, nombre de usuario, contraseña y rol | SC-04 | El nuevo usuario queda registrado en el sistema con los datos indicados y puede autenticarse con las credenciales creadas | El usuario no queda registrado, o no puede autenticarse con las credenciales proporcionadas |
| ACP-09 | Al completar el arranque del sistema tras una caída, se envía notificación in-app a la administradora informando el restablecimiento | SC-05 | La administradora recibe notificación in-app al completarse el health-check de arranque | La notificación no se envía, se envía con retraso después del arranque, o se envía a un usuario incorrecto |
| ACP-10 | La administradora puede crear una reserva retroactiva (con fecha de inicio en el pasado) cuando no existe conflicto | SC-06 | La reserva retroactiva queda registrada en el sistema con es_retroactiva = verdadero y estado = retroactiva | El sistema rechaza la reserva retroactiva sin conflicto real o no la marca como retroactiva |
| ACP-11 | Antes de confirmar una reserva retroactiva con conflicto, el sistema muestra alerta informativa; la administradora puede confirmar de todas formas | SC-07 | El sistema muestra la alerta de conflicto, y al confirmar la administradora, registra la reserva retroactiva sin bloqueo | El sistema bloquea definitivamente la confirmación de la reserva retroactiva cuando hay conflicto |
| ACP-12 | El health-check automático al arrancar detecta la operatividad del sistema y envía notificación in-app a la administradora | SE-04 | La notificación de restablecimiento se genera automáticamente como resultado del proceso de arranque del sistema, sin acción manual | La notificación requiere acción manual, o el sistema arranca sin notificar a la administradora |
| ACP-13 | La alerta de conflicto en reserva retroactiva es informativa: no bloquea la creación de la reserva | SE-05 | La administradora puede confirmar la reserva retroactiva aún cuando el sistema detecta conflicto; la alerta es solo informativa | El sistema impide definitivamente la creación de la reserva retroactiva cuando hay conflicto |
| ACP-14 | La regla de exclusividad de solapamiento horario aplica a la administradora del mismo modo que al resto de usuarios | SE-06 | Si la administradora intenta crear una reserva (retroactiva o no) que solapa con su propia reserva activa, el sistema la bloquea | El sistema permite que la administradora tenga dos reservas activas propias con solapamiento |

---

### Actor AC-03 — Gerente general

| ID     | Criterio | Escenario BDD | Condición de cumplimiento | Condición de fallo |
|--------|----------|---------------|--------------------------|-------------------|
| ACP-15 | Las reservas para reuniones con cliente externo o directorio se clasifican automáticamente como prioritarias | SC-08 | Al crear la reserva indicando tipo cliente externo o directorio, el tipo_prioridad queda como prioritaria sin acción adicional | El sistema crea la reserva como estándar aunque el tipo de reunión sea cliente externo o directorio |
| ACP-16 | Solo el rol gerente general puede marcar manualmente una reunión interna como urgente | SC-09 | Al usar la función de marcar urgente, la reserva pasa a tipo_prioridad = prioritaria; la función no está disponible para otros roles | Un usuario con rol empleado_general o administradora puede marcar una reserva como urgente |
| ACP-17 | Cuando la gerente crea una reserva prioritaria en una sala ocupada por una reserva estándar, el sistema ejecuta el desplazamiento automáticamente | SC-10 | La reserva estándar queda cancelada, la reserva prioritaria queda activa, y el usuario desplazado recibe notificación in-app con razón explícita | El desplazamiento no ocurre, o requiere aprobación manual, o el usuario desplazado no recibe notificación |
| ACP-18 | Cuando una reserva prioritaria desplaza una reserva activa, el usuario desplazado recibe notificación in-app con razón y alternativa (si existe) | SE-07 | La notificación al usuario desplazado incluye razón explícita; si hay sala disponible en el mismo horario, la incluye; si no hay, la notificación se envía igualmente | La notificación no incluye razón, o incluye una alternativa inexistente, o no se envía |
| ACP-19 | Ante empate de prioridad máxima, el sistema mantiene la reserva más antigua y rechaza la segunda de forma automática sin intervención de la administradora | SE-08 | La reserva con menor timestamp_creacion se mantiene activa; la segunda queda rechazada; la administradora no interviene | El sistema no resuelve el empate automáticamente, o mantiene la reserva incorrecta, o requiere intervención manual |
| ACP-20 | El solicitante de la segunda reserva en empate de prioridad recibe notificación in-app con rechazo y alternativa (si existe) | SE-08 | El solicitante de la segunda reserva recibe notificación in-app con el rechazo y, si hay sala alternativa disponible en el mismo horario, la incluye | El solicitante no recibe notificación, o la notificación no indica que fue rechazado por empate |
| ACP-21 | La regla de exclusividad de solapamiento horario aplica a la gerente general del mismo modo que al resto de usuarios | SE-09 | Si la gerente intenta crear una reserva que solapa con su propia reserva activa, el sistema la bloquea | El sistema permite que la gerente tenga dos reservas activas propias con solapamiento |

---

## Criterios de Aceptación Globales

(Criterios que aplican a todos los actores o al sistema en general.)

| ID     | Criterio | Escenario BDD | Condición de cumplimiento | Condición de fallo |
|--------|----------|---------------|--------------------------|-------------------|
| ACP-22 | El sistema resuelve todos los conflictos de prioridad de forma automática, sin intervención de la administradora | SC-10, SE-07, SE-08 | Los desplazamientos y rechazos por prioridad ocurren sin que la administradora deba aprobar o confirmar nada | El sistema detiene el flujo esperando aprobación de la administradora para resolver un conflicto de prioridad |
| ACP-23 | Todas las notificaciones del sistema se entregan exclusivamente a través del canal in-app; no se usan correo electrónico ni push externo | SC-02, SC-05, SE-01, SE-02, SE-04, SE-07, SE-08 | Las notificaciones son visibles al acceder a la aplicación; no se envían correos ni notificaciones push externas | El sistema envía correos o notificaciones push externas por cualquier evento |

---

## Trazabilidad inversa

| Escenario BDD | Criterio(s) de aceptación |
|---------------|--------------------------|
| SC-01 | ACP-01 |
| SC-02 | ACP-02, ACP-23 |
| SC-03 | ACP-03 |
| SC-04 | ACP-08 |
| SC-05 | ACP-09, ACP-12 |
| SC-06 | ACP-10 |
| SC-07 | ACP-11, ACP-13 |
| SC-08 | ACP-15 |
| SC-09 | ACP-16 |
| SC-10 | ACP-17, ACP-22 |
| SE-01 | ACP-04, ACP-05 |
| SE-02 | ACP-06 |
| SE-03 | ACP-07 |
| SE-04 | ACP-12 |
| SE-05 | ACP-13 |
| SE-06 | ACP-14 |
| SE-07 | ACP-18, ACP-22 |
| SE-08 | ACP-19, ACP-20 |
| SE-09 | ACP-21 |
