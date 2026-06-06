# Decisiones y Trade-offs — MeetingApp

*Como se aprendió en Contador_Calorias: documentar condiciones de revisión en cada decisión de diseño. Una decisión sin condición de revisión envejece sin actualizarse.*

---

## Decisión 1: Locking pesimista vs. optimista para detección de solapamientos

**El problema**: dos usuarios intentan reservar la misma sala al mismo tiempo. Ambos leen "disponible". Ambos insertan. Resultado: doble booking.

**Opción A: Locking pesimista (`SELECT FOR UPDATE`)**
- ✅ Garantía absoluta de no doble booking
- ✅ Simple de entender y debuggear: si el lock está tomado, la segunda transacción espera
- ✅ Nativo en PostgreSQL, sin librerías adicionales
- ❌ La segunda transacción espera (latencia) hasta que la primera termine
- ❌ Con muchos usuarios concurrentes en la misma sala, se forma una cola de espera

**Opción B: Locking optimista (versión + retry)**
- ✅ Sin bloqueo: las transacciones no esperan entre sí
- ✅ Mejor throughput bajo alta concurrencia
- ❌ Requiere lógica de retry en la aplicación cuando la versión no coincide
- ❌ Bajo alta concurrencia en la misma sala, muchos reintentos degradan la experiencia

**Decisión tomada**: Opción A — locking pesimista.

**Criterio desempate**: con 25 usuarios y 3 salas, la probabilidad de dos reservas simultáneas en la misma sala es muy baja. El locking pesimista es más simple y más seguro. La espera de <100ms es imperceptible para el usuario.

**Condición para reconsiderar**: si el sistema se expande a cientos de usuarios y hay colas visibles de espera en el endpoint de reserva (latencia > 1s en P95), migrar a locking optimista con retry.

---

## Decisión 2: Transaccionalidad de las notificaciones

**El problema**: ¿las notificaciones deben ocurrir dentro de la transacción de reserva, o después?

**Opción A: Notificación dentro de la transacción**
- ✅ Si la transacción falla, la notificación también se revierte — consistencia garantizada
- ✅ No hay notificaciones "fantasma" de operaciones que nunca completaron
- ❌ Si el INSERT de la notificación falla, toda la operación de reserva falla — coupling innecesario
- ❌ La transacción es más larga (más tiempo con el lock tomado)

**Opción B: Notificación con Transactional Outbox**
- ✅ La notificación se registra como "pendiente" dentro de la transacción; un proceso separado la envía
- ✅ Desacopla el éxito de la reserva del éxito de la notificación
- ❌ Mayor complejidad: necesita proceso background (worker) + tabla outbox
- ❌ Las notificaciones pueden llegar con delay (segundos) en lugar de instantáneamente

**Opción C: Notificación simple dentro de la transacción (recomendado para V1)**
- INSERT en `notificaciones_inapp` dentro del `BEGIN...COMMIT` de la reserva
- La notificación es part del mismo commit — si la reserva falla, no hay notificación

**Decisión tomada**: Opción C — notificación dentro de la misma transacción, sin outbox.

**Criterio desempate**: con notificaciones solo in-app (no correo, no push externo), el INSERT en una tabla local es instantáneo y no puede fallar independientemente del resto de la transacción. Si falla, toda la reserva falla — y es correcto no notificar de una reserva que no se creó.

**Condición para reconsiderar**: si se implementa DF-02 (notificaciones por correo o push externo), migrar al patrón Transactional Outbox para desacoplar el éxito de la reserva del éxito del envío externo.

---

## Decisión 3: JWT stateless vs. sesiones server-side

**El problema**: ¿cómo gestionar la sesión autenticada de los usuarios?

**Opción A: JWT stateless**
- ✅ Sin estado en el servidor — fácil de escalar horizontalmente
- ✅ No requiere Redis ni tabla de sesiones
- ❌ No se puede revocar un token antes de que expire (ej: si se cambia el rol de un usuario)
- ❌ Si el secreto se compromete, todos los tokens son vulnerables

**Opción B: Sesiones server-side (con Redis o DB)**
- ✅ Revocación inmediata (logout, cambio de contraseña, cambio de rol)
- ✅ El token es solo un ID aleatorio — sin información sensible
- ❌ Requiere almacenamiento de sesiones (más infraestructura)
- ❌ Stateful — complica el scaling horizontal

**Decisión tomada**: Opción A — JWT con access token de corta duración (15-30 min) + refresh token.

**Criterio desempate**: para 25 usuarios con roles que cambian raramente, el período máximo de inconsistencia (hasta que expire el access token) es aceptable. La complejidad de Redis + gestión de sesiones no se justifica.

**Condición para reconsiderar**: si se implementa DF-05 (gestión de usuarios por el propio usuario con cambio de contraseña autónomo), o si hay un incidente de seguridad donde sea necesario revocar tokens inmediatamente, migrar a sesiones server-side o implementar una lista de tokens revocados en Redis.

---

## Decisión 4: Reservas retroactivas — alerta informativa vs. bloqueante

**El problema**: cuando la administradora registra una reserva con hora pasada y hay conflicto con otra reserva ya registrada, ¿el sistema le permite confirmar de todas formas o lo bloquea?

**Opción A: Alerta informativa (la admin decide)**
- ✅ La administradora tiene el contexto completo — sabe cuál reserva es "real"
- ✅ Permite resolver ambigüedades del fallback (dos personas dicen haber coordinado la misma sala)
- ❌ Puede crear estados inconsistentes si la admin confirma sin verificar

**Opción B: Alerta bloqueante (el sistema impide la confirmación)**
- ✅ Garantiza que no haya dos reservas para la misma sala en el mismo horario en el registro histórico
- ❌ Si hay un conflicto legítimo a resolver (ej: el sistema tiene un registro equivocado), la admin queda bloqueada

**Decisión tomada**: Opción A — alerta informativa, documentada en ACP-11, ACP-13, EP-05.

**Criterio desempate**: durante una caída, el fallback (WhatsApp con Ana) es manual y puede generar ambigüedades. La administradora es la única que tiene el contexto completo para resolverlas. Bloquearla sería más perjudicial que permitirle ejercer su criterio.

**Condición para reconsiderar**: si en uso real se observan errores frecuentes de registros retroactivos incorrectos que afectan la integridad del historial.

---

## Decisión 5: Monolito vs. separar el servicio de notificaciones

**El problema**: ¿las notificaciones deben ser un servicio separado o parte del monolito?

**Opción A: Notificaciones en el monolito (mismo proceso)**
- ✅ Sin latencia de red entre la reserva y la notificación
- ✅ Sin gestión de otro servicio
- ❌ Si las notificaciones crecen en complejidad (múltiples canales), el monolito crece

**Opción B: Microservicio de notificaciones**
- ✅ Escala independientemente si el volumen de notificaciones crece
- ✅ Añadir nuevos canales (DF-02) no toca el servicio de reservas
- ❌ Comunicación asincrónica entre servicios (message queue) — complejidad para 25 usuarios

**Decisión tomada**: Opción A — notificaciones en el monolito, pero bien encapsuladas en su propio módulo.

**Criterio desempate**: YAGNI. *Como se aprendió en Contador_Calorias: no se implementa para necesidades futuras no confirmadas.* La separación puede hacerse en el futuro si se añaden canales externos (DF-02) y la complejidad lo justifica.

---

## Decisión 6: Timestamp de creación en BD vs. en aplicación

**El problema**: el `timestamp_creacion` de cada reserva determina el desempate de prioridad (RN-04). ¿Quién lo genera — la aplicación o la base de datos?

**Opción A: Generado por la BD (`DEFAULT NOW()`)**
- ✅ El reloj de la BD es único y consistente — no hay skew entre servidores de aplicación
- ✅ El timestamp es inmutable: la aplicación no puede falsificarlo (RN-06)
- ❌ La aplicación no conoce el timestamp exacto hasta después del INSERT

**Opción B: Generado por la aplicación**
- ✅ La aplicación puede incluir el timestamp en eventos de dominio antes del INSERT
- ❌ Con múltiples instancias de la aplicación, los relojes pueden tener skew de milisegundos
- ❌ La aplicación podría enviar un timestamp falso (seguridad)

**Decisión tomada**: Opción A — `DEFAULT NOW()` en PostgreSQL.

**Criterio desempate**: la inmutabilidad es crítica (RN-06). El timestamp de creación debe ser confiable para resolver empates justos. Un valor generado en la BD es el más confiable.

---

## Lecciones transferibles

1. **El locking pesimista es la opción segura cuando la concurrencia es baja y la consistencia es crítica**. Para sistemas de booking (reservas de hotel, tickets de avión, turnos médicos), `SELECT FOR UPDATE` es la solución canónica.

2. **La transaccionalidad de los efectos secundarios (notificaciones, auditoría) debe decidirse explícitamente**. No como afterthought — la decisión de incluirlos o no en la transacción tiene consecuencias de consistencia.

3. **Los roles y permisos deben estar centralizados y verificados server-side siempre**. La UI que oculta botones es UX, no seguridad.

4. **La complejidad del sistema de fallback ya define la complejidad de recuperación**. Que el fallback sea "llamar a Ana por WhatsApp" significa que la recuperación también depende de Ana — diseñar en consecuencia (retroactivas, notificación de restablecimiento).
