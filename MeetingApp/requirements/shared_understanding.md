# Shared Understanding Document — 010 Discovery
Fecha: 2026-06-01
Estado: APROBADO POR CLIENTE

## Propósito del Sistema

El sistema resuelve un problema concreto de coordinación interna: en una empresa de 25 personas con 3 salas de reunión disponibles, la asignación de salas se hacía informalmente a través de WhatsApp con la administradora (Ana), generando conflictos, falta de visibilidad y dependencia de un intermediario para cada reserva. El nuevo sistema elimina ese intermediario para el caso normal y permite que cada empleado gestione sus propias reservas de forma autónoma.

El sistema introduce además una capa de prioridad para proteger las reuniones más críticas del negocio — aquellas con clientes externos o miembros del directorio — frente a reservas ordinarias. Esta protección opera automáticamente, sin que la administradora deba intervenir en la resolución de conflictos. El gerente general puede adicionalmente marcar reuniones internas como urgentes cuando lo considera necesario, ejerciendo así control sobre la priorización.

La administradora conserva un rol operativo clave: crea y gestiona los usuarios del sistema, actúa como punto de coordinación manual (fallback) si el sistema cae, y puede registrar retroactivamente las reservas tomadas durante una caída para mantener el historial íntegro.

En todos los casos, la comunicación del sistema con los usuarios ocurre a través de notificaciones dentro de la propia aplicación. No se utilizan correo electrónico, mensajes push externos ni integración con calendarios de terceros.

## Actores y sus Necesidades

### Empleado general (A-01)
- **Descripción:** Cualquier miembro de la empresa sin rol privilegiado. Necesita reservar salas para reuniones internas sin depender de intermediarios.
- **Necesidades principales:**
  - Ver qué salas están disponibles para un horario dado antes de reservar (OV-01)
  - Reservar una sala de forma autónoma, sin pasar por Ana (OV-02)
  - Recibir confirmación inmediata tras hacer una reserva (OV-03)
  - Poder reservar con poca anticipación, incluso 1–2 horas antes, sin tiempo mínimo obligatorio (OV-04)
  - Cuando su reserva es cancelada por desplazamiento de prioridad: recibir notificación in-app con la razón y, si existe, una sala alternativa para el mismo horario (OV-05)
  - Cuando no hay disponibilidad: recibir un mensaje con los próximos horarios disponibles (OV-06)

### Administradora (A-02)
- **Descripción:** Ana Rodríguez, responsable operativa del sistema. Gestiona usuarios, actúa como fallback durante caídas y registra reservas retroactivas tras la recuperación.
- **Necesidades principales:**
  - Que el sistema resuelva conflictos de prioridad automáticamente, sin requerir su aprobación (OV-07)
  - Que todas las partes afectadas por una cancelación sean notificadas in-app sin intervención manual (OV-08)
  - Recibir notificación in-app cuando el sistema se restablece tras una caída (OV-09)
  - Poder crear reservas con hora pasada para registrar las coordinadas por WhatsApp durante la caída (OV-10)
  - Al crear una reserva retroactiva, ver alerta de conflicto antes de confirmar si ya existe otra reserva para esa sala y horario (OV-11)

### Gerente general (A-03)
- **Descripción:** Laura Jiménez, gerente de la empresa. Tiene capacidades de priorización exclusivas que ningún otro rol posee.
- **Necesidades principales:**
  - Que las reservas de reuniones con clientes externos o con el directorio sean automáticamente protegidas como prioritarias (OV-12)
  - Poder marcar manualmente una reunión interna como urgente — capacidad exclusiva del rol gerente (OV-13)
  - Que el sistema desplace automáticamente reservas de menor prioridad para acomodar sus reservas críticas, sin gestión manual (OV-14)
  - Que las personas desplazadas reciban notificación in-app con razón explícita y oferta de alternativa si existe disponibilidad (OV-15)

## Lo que el Sistema Hace

1. El sistema permite a cualquier usuario autenticado consultar qué salas (de las 3 disponibles) están libres para un horario dado, mostrando nombre, capacidad y equipamiento de cada sala.
2. El sistema permite a cualquier usuario autenticado reservar una sala disponible para un horario futuro, sin tiempo mínimo de anticipación, recibiendo confirmación inmediata.
3. El sistema impide que un mismo usuario tenga más de una reserva activa con solapamiento de horario (regla de exclusividad por usuario). Cualquier solapamiento parcial activa este bloqueo.
4. El sistema clasifica automáticamente como prioritarias las reservas de reuniones con clientes externos o con miembros del directorio.
5. El sistema permite al gerente general marcar manualmente una reunión interna como urgente, elevando su prioridad. Esta capacidad es exclusiva del rol gerente.
6. Cuando una reserva prioritaria requiere una sala ocupada por una reserva de menor prioridad, el sistema cancela automáticamente la reserva de menor prioridad (desplazamiento) y notifica in-app al afectado con la razón y, si existe, una sala alternativa disponible para el mismo horario.
7. Cuando dos reservas del mismo nivel de prioridad máxima compiten por la misma sala y horario, el sistema resuelve el conflicto automáticamente: gana la reserva creada primero (orden cronológico de creación). No requiere intervención de la administradora.
8. El sistema notifica in-app al empleado cuando no hay salas disponibles para su horario solicitado, sugiriendo los próximos horarios con disponibilidad.
9. El sistema notifica in-app a la administradora cuando se restablece tras una caída.
10. El sistema permite a la administradora crear reservas con hora de inicio en el pasado (reservas retroactivas) para registrar las ocupaciones coordinadas manualmente durante una caída. Antes de confirmar, el sistema muestra alerta si existe conflicto de horario con otra reserva ya registrada.
11. La administradora es la única que puede crear y gestionar cuentas de usuario en el sistema. La autenticación es por usuario y contraseña, sin integración con directorios externos.

## Contradicciones Resueltas

| Contradicción | Resolución acordada |
|---------------|---------------------|
| C-01: Posible conflicto sobre si la regla de exclusividad (un usuario no puede tener más de una reserva activa en el mismo horario) aplica también al rol gerente | La gerente (S-03) aceptó explícitamente: "La regla me parece bien para todos los roles, incluido el mío." La regla aplica de forma uniforme a todos los actores del sistema. |

## Aprobación del Cliente

Estado: APROBADO POR CLIENTE
Fecha de aprobación: 2026-06-01
Registro: El cliente aprobó explícitamente los 4 artefactos del Discovery con el mensaje "Apruebo los artefactos, continúa con el 020".
