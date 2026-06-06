# Scope Boundaries — 010 Discovery
Fecha: 2026-06-01

## Qué NO hará el sistema en esta etapa

| ID    | Exclusión | Razón | Fuente |
|-------|-----------|-------|--------|
| EX-01 | Sin integración con calendarios externos (Google Calendar, Outlook u otros) | Decisión explícita del cliente: el sistema gestiona sus propias reservas sin sincronización externa. | Transcript — S-01, V-04 resuelto en iteración 3 |
| EX-02 | Sin notificaciones por correo electrónico ni push externo | Canal único definido: notificaciones in-app. Correo y push fueron descartados explícitamente. | Transcript — S-01, AM-01 resuelta en iteración 1 |
| EX-03 | Sin filtrado de salas por atributos (capacidad o equipamiento) | El cliente decidió mostrar siempre las 3 salas disponibles con sus atributos; el usuario elige visualmente. No hay motor de filtrado. | Transcript — S-02, V-02 resuelto en iteración 2 |
| EX-04 | Sin autenticación federada ni directorio externo (LDAP, Active Directory, SSO) | Autenticación propia por usuario y contraseña; usuarios creados y gestionados por la administradora dentro del sistema. | Transcript — S-01, V-01 / V-04 resueltos en iteraciones 2–3 |
| EX-05 | Sin aprobación manual de reservas por parte de la administradora | El sistema resuelve conflictos de prioridad de forma automática. La administradora no actúa como aprobadora de reservas ordinarias. | Transcript — S-01, OV-07 |
| EX-06 | Sin reportes ni historial de uso de salas accesible por los usuarios | No fue mencionado como objetivo de valor por ningún stakeholder. Queda fuera del scope actual. | Límite implícito del scope acordado |
| EX-07 | Sin gestión de solicitudes de reserva pendientes de confirmación (flujo de aprobación) | El modelo es de reserva directa e inmediata: confirmación automática o rechazo automático. No existe estado "pendiente de aprobación". | Límite implícito del scope acordado |

## Qué queda diferido (posibles futuras fases)

| ID    | Capacidad diferida | Condición para incluir |
|-------|--------------------|----------------------|
| DF-01 | Filtrado de salas por capacidad o equipamiento | Si la empresa crece y la elección visual entre 3 salas ya no es suficiente, o si se agregan más salas al inventario. |
| DF-02 | Notificaciones por correo o push externo | Si los usuarios demandan confirmaciones fuera de la aplicación, o si la empresa adopta herramientas de comunicación que lo justifiquen. |
| DF-03 | Integración con calendarios externos | Si la empresa adopta una plataforma de productividad corporativa (Google Workspace, Microsoft 365) y requiere sincronización bidireccional. |
| DF-04 | Reportes de ocupación y uso de salas | Si la administradora o la gerencia requieren métricas de utilización para optimizar los espacios. |
| DF-05 | Gestión de usuarios por el propio usuario (auto-registro, cambio de contraseña autónomo) | Si la carga operativa de gestión de usuarios por Ana resulta excesiva o si la empresa crece. |

## Restricciones activas

| Tipo | Restricción | Impacto |
|------|-------------|---------|
| Organizacional | La empresa tiene exactamente 3 salas de reunión. El sistema no contempla más ni menos. | El modelo de "mostrar las 3 disponibles" es fijo; si cambia el inventario de salas, se requiere revisión del scope. |
| Organizacional | La empresa tiene 25 personas como tamaño actual. El sistema no está diseñado para escala masiva. | No hay requerimientos de rendimiento para grandes volúmenes de usuarios concurrentes en esta fase. |
| Operacional | La administradora (Ana) es el único punto de fallback durante caídas del sistema y la única que puede crear usuarios. | Dependencia crítica de un rol individual; no hay delegación de estas capacidades a otros usuarios. |
| Tecnológica | Sin integración con sistemas externos de autenticación o calendarios. | El sistema es una aplicación autónoma; cualquier consistencia con herramientas externas es responsabilidad manual del usuario. |
| De proceso | El desempate de reservas de igual prioridad se resuelve por orden cronológico de creación, de forma automática y sin intervención humana. | El sistema debe registrar el timestamp exacto de cada reserva para aplicar esta regla de forma confiable. |
