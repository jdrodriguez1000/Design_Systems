# Error & Exception Policy — 020 Specification
Fecha: 2026-06-01
Estado: DRAFT
Basado en: specification/spec_analysis_report.md

## Políticas por Actor

### Actor AC-01 — Empleado general

| ID     | Escenario de error (EE-xx) | Causa | Mensaje al usuario | Reintento | Bloqueo | Acción alternativa | Escenario BDD relacionado |
|--------|---------------------------|-------|-------------------|-----------|---------|-------------------|--------------------------|
| EP-01  | EE-01 — Reserva activa cancelada por desplazamiento de prioridad | Una reserva de mayor prioridad reclama la misma sala en el mismo horario que la reserva del empleado | "Tu reserva para [nombre sala] el [fecha] de [hora inicio] a [hora fin] fue cancelada porque fue reasignada por una reunión prioritaria. [Si existe alternativa: La sala [nombre sala alternativa] está disponible para ese horario.]" | no | no | Enviar notificación in-app al empleado afectado con razón explícita y sala alternativa (si existe); si no hay alternativa, la notificación se envía igualmente sin oferta |
| EP-02  | EE-02 — No hay salas disponibles para el horario solicitado | Todas las salas están ocupadas en el intervalo solicitado | "No hay salas disponibles para el horario solicitado. Los próximos horarios disponibles son: [lista de horarios con al menos una sala libre]." | no | sí (bloquea la confirmación de reserva) | Mostrar lista de próximos horarios disponibles para que el empleado pueda seleccionar uno alternativo |
| EP-03  | EE-07 — Intento de reserva con solapamiento del mismo usuario (exclusividad) | El usuario ya tiene una reserva activa que se solapa en horario con la nueva solicitud | "No puedes crear esta reserva porque ya tienes una reserva activa que se solapa con este horario: [nombre sala] el [fecha] de [hora inicio] a [hora fin]." | no | sí (bloquea la nueva reserva) | Ninguna; el usuario debe cancelar o modificar su reserva existente o elegir un horario sin solapamiento |

---

### Actor AC-02 — Administradora

| ID     | Escenario de error (EE-xx) | Causa | Mensaje al usuario | Reintento | Bloqueo | Acción alternativa | Escenario BDD relacionado |
|--------|---------------------------|-------|-------------------|-----------|---------|-------------------|--------------------------|
| EP-04  | EE-03 — Caída total del sistema (notificación de restablecimiento) | El sistema ha estado inaccesible y acaba de completar el arranque con health-check exitoso | "El sistema se ha restablecido correctamente. Ya puedes retomar el uso normal. Recuerda registrar las reservas coordinadas por WhatsApp durante la caída." | no | no | Enviar notificación in-app automática a la administradora al completar el arranque; no requiere acción de ningún usuario |
| EP-05  | EE-04 — Conflicto detectado al registrar reserva retroactiva | Al crear una reserva retroactiva, el sistema detecta solapamiento con otra reserva ya registrada en la misma sala y horario | "Atención: ya existe una reserva para [nombre sala] en el horario [hora inicio]–[hora fin] el [fecha]. Puedes confirmar esta reserva retroactiva de todas formas si corresponde a lo coordinado durante la caída." | no | no (alerta informativa, no bloqueante) | Mostrar alerta de conflicto antes de confirmar; la administradora puede confirmar la reserva retroactiva o cancelar la operación |

---

### Actor AC-03 — Gerente general

| ID     | Escenario de error (EE-xx) | Causa | Mensaje al usuario | Reintento | Bloqueo | Acción alternativa | Escenario BDD relacionado |
|--------|---------------------------|-------|-------------------|-----------|---------|-------------------|--------------------------|
| EP-06  | EE-05 — Reserva prioritaria del gerente desplaza reserva activa | El gerente crea una reserva prioritaria para una sala ocupada por una reserva de menor prioridad | (Mensaje al usuario desplazado, no al gerente) "Tu reserva para [nombre sala] el [fecha] de [hora inicio] a [hora fin] fue cancelada porque fue reasignada por una reunión prioritaria. [Si existe alternativa: La sala [nombre sala alternativa] está disponible para ese horario.]" | no | no | Cancelar la reserva de menor prioridad automáticamente; enviar notificación in-app al usuario desplazado con razón explícita y alternativa (si existe); confirmar la reserva prioritaria del gerente |
| EP-07  | EE-06 — Empate de prioridad máxima: segunda reserva rechazada | Dos reservas con igual nivel de prioridad máxima compiten por la misma sala y horario; gana la creada primero | (Mensaje al solicitante de la segunda reserva) "Tu reserva para [nombre sala] el [fecha] de [hora inicio] a [hora fin] no pudo ser confirmada porque ya existe una reserva prioritaria para ese horario que fue creada antes. [Si existe alternativa: La sala [nombre sala alternativa] está disponible para ese horario.]" | no | sí (bloquea la segunda reserva) | Rechazar la segunda reserva automáticamente; enviar notificación in-app al solicitante con rechazo y sala alternativa (si existe); la administradora no interviene |

---

## Políticas Globales

(Errores no atribuibles a un actor específico — reglas de sistema que aplican a todos los actores.)

| ID     | Escenario de error (EE-xx) | Causa | Mensaje al usuario | Reintento | Bloqueo | Acción alternativa |
|--------|---------------------------|-------|-------------------|-----------|---------|-------------------|
| EP-08  | EE-07 — Solapamiento por exclusividad del usuario (SG-01) — aplicado a todos los roles | El usuario (cualquier rol) ya tiene una reserva activa que se solapa en horario con la nueva solicitud | "No puedes crear esta reserva porque ya tienes una reserva activa que se solapa con este horario: [nombre sala] el [fecha] de [hora inicio] a [hora fin]." | no | sí (bloquea la nueva reserva) | Ninguna; el usuario debe cancelar o modificar su reserva existente o elegir un horario sin solapamiento |
| EP-09  | EE-08 — Restablecimiento del sistema tras caída (SG-02) | El sistema completa el proceso de arranque y el health-check automático confirma operatividad | "El sistema se ha restablecido correctamente. Ya puedes retomar el uso normal." (notificación destinada a la administradora) | no | no | Enviar notificación in-app automática a la administradora; el resto de usuarios retoma el uso normal sin notificación específica |

---

## Resoluciones de ítems PENDIENTE del 010

| Ítem original (EE-xx) | Pregunta que estaba pendiente | Resolución del governor | Política aplicada (EP-xx) |
|-----------------------|------------------------------|------------------------|--------------------------|
| EE-03 / EE-08 (SF-03 / SG-02) | ¿Cómo detecta el sistema que se ha restablecido? ¿Proceso automático de health-check, arranque manual, o evento externo? | Opción A — Health-check automático. Al arrancar, el sistema detecta que está operativo y notifica a la administradora. Sin dependencias externas ni coordinación humana. | EP-04, EP-09 |
| EE-06 (SF-06) | ¿Qué mensaje recibe el solicitante de la segunda reserva cuando pierde el empate de prioridad máxima? ¿Solo rechazo o rechazo + oferta de alternativa? | Opción B — Rechazo + oferta de alternativa. Consistente con el comportamiento estándar de desplazamiento (SF-01/SF-05). | EP-07 |
| EE-04 (SF-04) | Cuando la administradora ve la alerta de conflicto al registrar una reserva retroactiva, ¿puede confirmarla de todas formas o el sistema bloquea definitivamente? | Opción A — Informativa. La administradora puede confirmar aunque haya conflicto conocido. El sistema avisa, ella decide. | EP-05 |
