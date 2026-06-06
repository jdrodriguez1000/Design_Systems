# Domain Glossary — 010 Discovery
Fecha: 2026-06-01

## Términos del Dominio

| Término | Definición acordada | Sinónimos a evitar | Actor que lo usa |
|---------|--------------------|--------------------|-----------------|
| Reserva | Asignación de una sala a un usuario para un horario específico con inicio y fin definidos. Una reserva activa ocupa la sala durante todo ese intervalo. | Booking, cita, bloqueo | A-01, A-02, A-03 |
| Reserva activa | Reserva confirmada cuyo horario aún no ha concluido y que no ha sido cancelada. | Reserva vigente, reserva en curso | A-01, A-02, A-03 |
| Sala | Espacio físico de reunión identificado por nombre, capacidad (número de personas) y equipamiento disponible (proyector, videoconferencia, pizarra). | Sala de reuniones, cuarto, espacio | A-01, A-02, A-03 |
| Conflicto | Situación en que dos reservas se solapan en la misma sala, o un mismo usuario intenta tener más de una reserva activa en un mismo horario. Cualquier solapamiento parcial califica como conflicto. | Choque, colisión, doble reserva | A-01, A-02, A-03 |
| Desplazamiento | Cancelación automática de una reserva activa de menor prioridad para dar lugar a una reserva de mayor prioridad en la misma sala y horario. | Expulsión, reemplazo, override | A-02, A-03 |
| Prioridad | Nivel de importancia de una reserva que determina su resistencia al desplazamiento. Niveles: prioritaria (cliente externo, directorio, o marcada manualmente por gerente) y estándar. | Jerarquía, urgencia, nivel | A-02, A-03 |
| Reserva prioritaria | Reserva que no puede ser desplazada por otras reservas estándar. Se asigna automáticamente a reuniones con clientes externos o directorio, o manualmente por el gerente. | Reserva urgente, reserva crítica, reserva VIP | A-03 |
| Fallback | Procedimiento manual de reserva de salas a través de WhatsApp con la administradora, activado exclusivamente durante caída del sistema. | Plan B, alternativa, contingencia | A-01, A-02 |
| Notificación in-app | Alerta generada dentro de la propia aplicación y visible al usuario al acceder. Canal único de notificación del sistema; sin correo ni push externo. | Alerta, aviso, push, correo | A-01, A-02, A-03 |
| Reserva retroactiva | Reserva creada con una hora de inicio en el pasado, permitida exclusivamente a la administradora para registrar ocupaciones ocurridas durante una caída del sistema. | Reserva con hora pasada, reserva manual posterior | A-02 |
| Solapamiento | Coincidencia parcial o total entre los intervalos horarios de dos reservas. Cualquier solapamiento, incluso parcial, activa la regla de conflicto. | Superposición, traslape, coincidencia | A-01, A-02, A-03 |
| Rol | Perfil de permisos asignado a un usuario al momento de su creación. Roles definidos: empleado general y gerente. La administradora tiene capacidades adicionales propias de su función. | Perfil, tipo de usuario, nivel de acceso | A-02 |
| Empate de prioridad | Situación en que dos reservas con el mismo nivel de prioridad (máxima) compiten por la misma sala en el mismo horario. Se resuelve por orden cronológico: la reserva creada primero gana. | Tie, conflicto entre iguales | A-02, A-03 |

## Abreviaturas

| Abreviatura | Expansión | Contexto de uso |
|-------------|-----------|-----------------|
| S-01        | Stakeholder 01 — Ana Rodríguez (Administradora) | Reporte de análisis, trazabilidad |
| S-02        | Stakeholder 02 — Carlos Mendoza (Empleado general) | Reporte de análisis, trazabilidad |
| S-03        | Stakeholder 03 — Laura Jiménez (Gerente general) | Reporte de análisis, trazabilidad |
| A-01        | Actor del sistema: Empleado general | Artefactos de discovery |
| A-02        | Actor del sistema: Administradora | Artefactos de discovery |
| A-03        | Actor del sistema: Gerente general | Artefactos de discovery |
| OV          | Objetivo de Valor | Reporte de análisis |
| SF          | Escenario de Fallo | Reporte de análisis, failure_behavior.md |
