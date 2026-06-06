# Conceptos Avanzados — Contador de Calorías

## Introducción

Este documento aplica conceptos de sistemas distribuidos a una app que, en su forma actual, no es distribuida. El objetivo es educativo: entender estos conceptos en un contexto simple antes de aplicarlos a sistemas complejos.

---

## Teorema CAP aplicado

El teorema CAP establece que un sistema distribuido solo puede garantizar 2 de estas 3 propiedades simultáneamente:
- **C**onsistency (Consistencia): todos los nodos ven los mismos datos al mismo tiempo
- **A**vailability (Disponibilidad): toda petición recibe respuesta (no siempre con los datos más recientes)
- **P**artition Tolerance (Tolerancia a particiones de red): el sistema sigue funcionando si hay fallos de red entre nodos

### Aplicado a este sistema: CP (técnicamente no aplica, pero el razonamiento es válido)

La arquitectura actual evita el dilema CAP por completo: hay **un único nodo** (el navegador de Sofía). No hay red, no hay particiones posibles, no hay conflicto entre nodos.

Sin embargo, si se implementara DF-04 (sincronización entre dispositivos), el sistema enfrentaría el dilema:

**Si Sofía registra una comida en el celular mientras el laptop está offline**:
- **Opción CP**: bloquear el registro hasta que ambos dispositivos estén sincronizados (consistencia garantizada, disponibilidad sacrificada)
- **Opción AP**: permitir el registro offline y resolver el conflicto después (disponibilidad garantizada, consistencia eventual)

Para este dominio (calorías personales), **AP es la elección correcta**: es mejor que Sofía pueda registrar su almuerzo aunque esté sin conexión, aunque el laptop tarde unos minutos en sincronizarse. El costo de inconsistencia temporal es bajo; el costo de no poder registrar es alto (olvido de datos).

Ejemplos reales con AP: DynamoDB (Amazon), Cassandra, el carrito de compras de Amazon (famoso paper de Werner Vogels sobre consistencia eventual).

---

## Modelo de consistencia

**Consistencia del estado actual**: el sistema usa **consistencia fuerte** de facto — hay un único nodo, cada operación lee y escribe al mismo `localStorage`, no hay concurrencia. El estado es siempre el más reciente.

**Si hubiera múltiples dispositivos**: el modelo correcto sería **consistencia eventual**. Los cambios en un dispositivo se propagan al otro eventualmente (cuando hay conexión). Google Calendar usa este modelo: si creas un evento offline en el teléfono, eventualmente aparece en el navegador del laptop.

---

## Alta disponibilidad

**Disponibilidad actual**: 100% desde la perspectiva del sistema — la app está disponible siempre que el navegador esté abierto, sin dependencia de servidores externos.

**Punto de fallo real**: `localStorage` del navegador. Si el usuario limpia los datos del navegador (Configuración → Borrar datos de navegación), los datos del día desaparecen. No hay backup.

**Estrategia de mitigación sin backend**: exportar los datos del día a un archivo JSON descargable. Si algún día Sofía pide esta función (no la pidió), sería una mejora de alta disponibilidad de bajo costo.

---

## Tolerancia a fallos

### Circuit Breaker — No aplica en la versión actual
El Circuit Breaker sirve para proteger a un servicio de llamar repetidamente a un servicio caído. Sin red, no hay circuito que proteger.

### Retry Policy — Aplica para localStorage
`localStorage` puede fallar si el almacenamiento del navegador está lleno (límite de ~5MB). La política correcta es:
1. Intentar la operación
2. Si falla, mostrar mensaje de error claro al usuario (no reintentar silenciosamente)
3. Nunca perder datos silenciosamente

### Bulkhead Pattern — No aplica
El Bulkhead aísla fallos entre componentes del sistema para que un fallo en un servicio no derrumbe toda la app. Sin servicios independientes, no hay bulkheads que diseñar.

---

## ACID vs BASE

### Estado actual (localStorage): ni ACID ni BASE estrictamente

`localStorage` no ofrece transacciones. Una operación de escritura (`setItem`) es atómica a nivel de clave, pero no hay forma de hacer una operación atómica que involucre múltiples claves (ej: actualizar las entradas y la fecha al mismo tiempo).

**Implicación práctica**: si el navegador se cierra exactamente mientras se está escribiendo en `localStorage` durante el reinicio diario, podría quedar un estado inconsistente (entradas borradas pero fecha no actualizada). Solución simple: escribir la nueva fecha **antes** de borrar las entradas — en el peor caso, el reinicio se aplica dos veces sin pérdida de datos.

### Si hubiera base de datos SQL (ej: para DF-04 con backend): ACID completo
Cada operación de negocio (agregar entrada, reinicio diario) sería una transacción atómica. El "reinicio diario" sería: `BEGIN; DELETE FROM entries WHERE user_id=? AND date=?; UPDATE last_reset SET date=TODAY WHERE user_id=?; COMMIT;` — o ambos ocurren, o ninguno.
