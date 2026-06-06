# Data Contracts — 020 Specification
Fecha: 2026-06-01
Estado: DRAFT
Basado en: specification/spec_analysis_report.md

## Entidades y sus Contratos

---

### Entidad EN-01 — Reserva

**Descripción:** Asignación de una sala a un usuario para un horario específico con inicio y fin definidos. Una reserva activa ocupa la sala durante todo ese intervalo. Es la entidad central del sistema.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| usuario | referencia a Usuario | — | sí | Debe ser un usuario registrado en el sistema | SC-02, SC-06, SC-08 |
| sala | referencia a Sala | — | sí | Debe ser una de las 3 salas del sistema | SC-02, SC-06, SC-08 |
| fecha_inicio | fecha y hora | DD/MM/AAAA HH:MM | sí | Debe ser una hora válida dentro del horario laboral (8:00–19:00); puede ser en el pasado si la crea la administradora (reserva retroactiva) | SC-02, SC-03, SC-06 |
| fecha_fin | fecha y hora | DD/MM/AAAA HH:MM | sí | Debe ser posterior a fecha_inicio; debe estar dentro del horario laboral | SC-02, SC-03, SC-06 |
| tipo_prioridad | lista de valores | — | sí | Valores permitidos: estándar, prioritaria. Asignado automáticamente o por marcado manual del gerente | SC-08, SC-09, SC-10 |
| tipo_reunion | lista de valores | — | no | Valores cuando aplica: reunión interna, cliente externo, directorio. Determina prioridad automática. | SC-08 |
| timestamp_creacion | fecha y hora | ISO 8601 | sí | Registrado automáticamente por el sistema al momento de creación; inmutable; usado para resolver empates de prioridad | SE-08 |
| estado | lista de valores | — | sí | Valores permitidos: activa, cancelada, retroactiva. Una reserva retroactiva es creada por la administradora con fecha_inicio en el pasado | SC-06, SC-07, SE-01, SE-07 |
| es_retroactiva | booleano | — | sí | Verdadero si fecha_inicio está en el pasado al momento de creación; solo permitido para usuarios con rol administradora | SC-06, SC-07 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-01 | Un usuario no puede tener más de una reserva activa con solapamiento horario (cualquier solapamiento parcial activa esta regla) | alta |
| RN-02 | Una sala no puede tener más de una reserva activa en el mismo intervalo horario (cualquier solapamiento parcial activa esta regla) | alta |
| RN-03 | Una reserva con tipo_prioridad = prioritaria no puede ser desplazada por una reserva estándar | alta |
| RN-04 | Cuando dos reservas de igual prioridad máxima compiten por la misma sala y horario, la reserva con menor timestamp_creacion gana (creada primero) | alta |
| RN-05 | Una reserva con es_retroactiva = verdadero solo puede ser creada por un usuario con rol administradora | alta |
| RN-06 | El sistema registra el timestamp_creacion automáticamente y no permite modificarlo después | alta |
| RN-07 | Una reserva con tipo_reunion = cliente externo o directorio recibe tipo_prioridad = prioritaria de forma automática | alta |

---

### Entidad EN-02 — Sala

**Descripción:** Espacio físico de reunión identificado por nombre, capacidad y equipamiento disponible. El sistema gestiona exactamente 3 salas; el inventario es fijo en esta fase.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| nombre | texto | — | sí | Nombre único entre las 3 salas (Azul, Verde, Roja) | SC-01, SC-02 |
| capacidad_personas | número entero | — | sí | Número de personas que admite la sala | SC-01 |
| equipamiento | lista de valores | — | sí | Valores permitidos por ítem: proyector, videoconferencia, pizarra. Una sala puede tener varios ítems. | SC-01 |
| disponible_en_horario | booleano calculado | — | no (calculado) | Calculado por el sistema en función de reservas activas en el horario consultado; no es un atributo persistido | SC-01, SC-02 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-08 | El sistema siempre muestra las 3 salas al consultar disponibilidad; no existe filtrado por atributos (EX-03) | media |
| RN-09 | El inventario de salas es fijo en 3; no se contempla agregar o eliminar salas en esta fase | alta |

---

### Entidad EN-03 — Usuario

**Descripción:** Persona autenticada con un rol definido que interactúa con el sistema. Los usuarios son creados y gestionados exclusivamente por la administradora. La autenticación es por nombre de usuario y contraseña propios del sistema.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| nombre_completo | texto | — | sí | Nombre de la persona tal como aparece en el sistema | SC-04 |
| nombre_usuario | texto | — | sí | Identificador único de inicio de sesión; no puede repetirse entre usuarios | SC-04 |
| contraseña | texto | — | sí | Establecida por la administradora al crear el usuario; no se almacena en claro | SC-04 |
| rol | referencia a Rol | — | sí | Debe ser uno de los roles definidos: empleado_general, gerente, administradora | SC-04 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-10 | Solo la administradora puede crear y gestionar cuentas de usuario | alta |
| RN-11 | El nombre_usuario debe ser único en el sistema | alta |
| RN-12 | No existe autenticación federada ni integración con directorios externos (EX-04) | alta |

---

### Entidad EN-04 — Rol

**Descripción:** Perfil de permisos asignado a un usuario al momento de su creación. Define qué acciones puede realizar en el sistema. Los roles son fijos y definidos por el negocio.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| tipo | lista de valores | — | sí | Valores permitidos: empleado_general, gerente, administradora | SC-04, SC-09 |
| puede_marcar_urgente | booleano | — | sí | Verdadero solo para el tipo gerente; falso para los demás | SC-09, SE-09 |
| puede_crear_usuarios | booleano | — | sí | Verdadero solo para el tipo administradora; falso para los demás | SC-04 |
| puede_crear_retroactiva | booleano | — | sí | Verdadero solo para el tipo administradora; falso para los demás | SC-06, SC-07 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-13 | La capacidad de marcar una reunión como urgente es exclusiva del rol gerente | alta |
| RN-14 | La capacidad de crear reservas retroactivas es exclusiva del rol administradora | alta |
| RN-15 | La regla de exclusividad de solapamiento horario aplica a todos los roles sin excepción | alta |

---

### Entidad EN-05 — Notificación in-app

**Descripción:** Alerta generada dentro de la propia aplicación y visible al usuario al acceder. Es el único canal de comunicación del sistema con los usuarios; no se usan correo electrónico ni push externo.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| tipo | lista de valores | — | sí | Valores permitidos: confirmacion_reserva, cancelacion_desplazamiento, rechazo_reserva, no_disponibilidad, restablecimiento_sistema, alerta_conflicto_retroactiva | SC-02, SC-05, SE-01, SE-02, SE-04, SE-05, SE-08 |
| mensaje | texto | — | sí | Texto concreto en lenguaje del dominio; no puede ser genérico | SE-01, SE-02, SE-07, SE-08 |
| destinatario | referencia a Usuario | — | sí | Usuario que debe recibir la notificación | SC-02, SC-05, SE-01 |
| timestamp | fecha y hora | ISO 8601 | sí | Momento en que el sistema genera la notificación | SC-05, SE-04 |
| sala_alternativa | referencia a Sala | — | no | Sala alternativa disponible en el mismo horario; presente solo en notificaciones de cancelacion_desplazamiento y rechazo_reserva cuando existe alternativa | SE-01, SE-07, SE-08 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-16 | El canal único de notificación es in-app; no se envían correos electrónicos ni notificaciones push externas (EX-02) | alta |
| RN-17 | Toda cancelación por desplazamiento genera al menos una notificación in-app al usuario afectado | alta |
| RN-18 | La oferta de sala alternativa en una notificación de cancelación solo se incluye si existe disponibilidad real en el mismo horario | alta |

---

### Entidad EN-06 — Conflicto

**Descripción:** Situación en que dos reservas se solapan en la misma sala, o un mismo usuario intenta tener más de una reserva activa en el mismo horario. Cualquier solapamiento, incluso parcial, califica como conflicto.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| sala_afectada | referencia a Sala | — | sí | La sala en la que se produce el solapamiento | SE-03, SE-05, SE-06 |
| horario_afectado_inicio | fecha y hora | DD/MM/AAAA HH:MM | sí | Inicio del intervalo en conflicto | SE-03, SE-05 |
| horario_afectado_fin | fecha y hora | DD/MM/AAAA HH:MM | sí | Fin del intervalo en conflicto | SE-03, SE-05 |
| reservas_en_conflicto | lista de referencias a Reserva | — | sí | Las 2 o más reservas que se solapan | SE-01, SE-03, SE-07, SE-08 |
| tipo_conflicto | lista de valores | — | sí | Valores: solapamiento_usuario (mismo usuario, diferente sala), solapamiento_sala (misma sala, diferente usuario), empate_prioridad (misma sala, mismo horario, igual prioridad máxima) | SE-03, SE-06, SE-08 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-19 | Un solapamiento parcial (incluso de 1 minuto) entre dos reservas en la misma sala o del mismo usuario activa la regla de conflicto | alta |
| RN-20 | Un conflicto entre reservas de diferente prioridad se resuelve por desplazamiento automático (genera EN-07) | alta |
| RN-21 | Un conflicto entre reservas de igual prioridad máxima se resuelve por timestamp_creacion (gana la más antigua) | alta |

---

### Entidad EN-07 — Desplazamiento

**Descripción:** Cancelación automática de una reserva activa de menor prioridad para dar lugar a una reserva de mayor prioridad en la misma sala y horario. El sistema ejecuta el desplazamiento sin intervención de la administradora.

#### Campos

| Campo | Tipo de dato (negocio) | Formato | Obligatorio | Validaciones de negocio | Escenario BDD relacionado |
|-------|------------------------|---------|-------------|------------------------|--------------------------|
| reserva_cancelada | referencia a Reserva | — | sí | La reserva de menor prioridad que fue cancelada | SE-01, SE-07 |
| reserva_entrante | referencia a Reserva | — | sí | La reserva de mayor prioridad que causó el desplazamiento | SC-10, SE-07 |
| motivo | texto | — | sí | Razón del desplazamiento en lenguaje del dominio (ej: "reasignada por reunión prioritaria") | SE-01, SE-07 |
| timestamp_desplazamiento | fecha y hora | ISO 8601 | sí | Momento en que el sistema ejecutó el desplazamiento | SE-01, SE-07 |

#### Reglas de negocio de la entidad

| ID    | Regla | Prioridad |
|-------|-------|-----------|
| RN-22 | El desplazamiento es automático; la administradora no interviene como aprobadora (EX-05) | alta |
| RN-23 | Un desplazamiento siempre genera una notificación in-app al usuario desplazado con razón explícita | alta |

---

## Relaciones entre Entidades

| ID    | Entidad A | Relación | Entidad B | Cardinalidad | Restricción de negocio |
|-------|-----------|----------|-----------|--------------|------------------------|
| RE-01 | Usuario | realiza | Reserva | 1..N | Un usuario no puede tener más de una reserva activa con solapamiento de horario (RN-01) |
| RE-02 | Reserva | ocupa | Sala | N..1 | Una sala solo puede tener una reserva activa por intervalo horario (RN-02) |
| RE-03 | Reserva | tiene | Rol | N..1 | El nivel de prioridad de la reserva está condicionado por el rol del usuario y el tipo de reunión (RN-07, RN-13) |
| RE-04 | Reserva | genera | Notificación in-app | 0..N | Toda cancelación por desplazamiento genera al menos una notificación al afectado (RN-17) |
| RE-05 | Desplazamiento | cancela | Reserva (menor prioridad) | 1..1 | Un desplazamiento cancela exactamente una reserva de menor prioridad (RN-22) |
| RE-06 | Administradora | gestiona | Usuario | 1..N | Solo la administradora puede crear y gestionar cuentas de usuario (RN-10) |
| RE-07 | Conflicto | puede generar | Desplazamiento | 0..1 | Un conflicto entre reservas de diferente prioridad genera desplazamiento; entre iguales se resuelve por timestamp (RN-20, RN-21) |
