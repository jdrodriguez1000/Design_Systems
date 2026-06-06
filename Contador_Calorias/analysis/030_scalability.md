# Estrategias de Escalabilidad — Contador de Calorías

## Contexto: la escalabilidad no es el reto de este sistema

Este es el documento más honesto del análisis: **la escalabilidad es irrelevante para la versión actual** de este sistema. Una app personal de un solo usuario corre en el navegador del usuario — no hay servidores que escalar ni carga concurrente que manejar.

Sin embargo, este documento tiene valor educativo: analiza *qué cambiaría* si el sistema evolucionara, y documenta los cuellos de botella que aparecerían.

---

## Escalabilidad actual (frontend-only)

La "carga" del sistema actual es:
- **1 usuario concurrente** (Sofía Martínez)
- **~10-30 operaciones por día** (registrar comidas)
- **Cómputo**: una suma de array de ~20 elementos

No hay nada que escalar. El límite no es de capacidad computacional sino de **la batería del celular de Sofía**.

---

## Cuello de botella del diseño actual

**El reinicio diario cliente-side**: la lógica de reinicio a medianoche solo se ejecuta cuando la app se abre. Si Sofía tiene la app abierta exactamente a medianoche, el reinicio ocurre en tiempo real vía `setInterval` o `setTimeout`. Si la app está cerrada, el reinicio ocurre la próxima vez que la abre.

Esto no es un cuello de botella de escala — es una limitación de diseño para un único usuario. Para una app multi-usuario, sería un problema serio (¿qué pasa si el usuario nunca abre la app?).

---

## Estimaciones de capacidad

| Recurso | Estimación | Límite conocido |
|---------|-----------|----------------|
| Entradas por día | ~5-20 | Sin límite real (localStorage tiene ~5-10 MB) |
| Tamaño de una entrada en JSON | ~100 bytes | — |
| Tamaño total del historial diario | ~1-2 KB máximo | Límite de localStorage: ~5 MB |
| Tiempo de renderizado | <1ms | No perceptible para el usuario |

**Conclusión**: localStorage tiene espacio para ~50,000 entradas diarias antes de llegar a su límite. Sofía nunca lo alcanzará.

---

## Si el sistema evolucionara: análisis de escalabilidad real

### Escenario: DF-04 — Sincronización entre dispositivos (implica añadir backend)

En este escenario, los desafíos de escalabilidad serían:

**Escalabilidad vertical vs horizontal**:
- Para ~1 usuario: un servidor pequeño (1 CPU, 512MB RAM) es suficiente de por vida
- Para ~10,000 usuarios: horizontal scaling con múltiples instancias de la API detrás de un load balancer
- Para ~1M usuarios (una app de dietas pública): se necesita sharding de la base de datos

**Estrategia de caché**:
- El historial del día es un candidato perfecto para caché por usuario en Redis con TTL de 24 horas
- La meta calórica cambia raramente — caché con invalidación explícita

**Base de datos bajo carga**:
- El patrón de acceso es simple: todas las queries filtran por `user_id + date`
- Un índice compuesto `(user_id, date)` en la tabla de entradas resuelve el 99% de las queries

**Auto-scaling**: para este dominio específico, el tráfico tiene un patrón predecible: picos después de desayuno, almuerzo y cena. Un sistema de auto-scaling podría usar estos picos históricos para escalar proactivamente.

---

## Lección transferible

El error más común en diseño de sistemas es diseñar para la escala que se *espera tener en 5 años* cuando se está construyendo para 1 usuario hoy. Netflix no nació con microservicios — empezó como un monolito en 2008 y migró gradualmente entre 2009 y 2016 conforme la escala lo exigió.

**La regla práctica**: diseña para 10x tu escala actual. Cuando llegas a ese límite, ya tienes los recursos (equipo, dinero, datos de uso real) para rediseñar con mejor información.
