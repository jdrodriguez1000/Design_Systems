# Decisiones y Trade-offs — Contador de Calorías

Este es el documento más importante para el aprendizaje. Cada decisión aquí tiene argumentos válidos en más de una dirección. No hay respuesta perfecta — hay una respuesta **correcta para este contexto**.

---

## Decisión 1: Frontend-only vs. Frontend + Backend

**El problema que forzó la decisión**: Sofía accede desde celular y computadora. ¿Los datos deben estar disponibles en ambos dispositivos?

**Opción A: Frontend-only con localStorage**
- ✅ Cero costo de infraestructura
- ✅ Funciona sin internet (de facto)
- ✅ Despliegue trivial
- ✅ Sin complejidad de auth
- ❌ Los datos no se sincronizan entre dispositivos
- ❌ Si limpia el caché, pierde el historial del día

**Opción B: Frontend + Backend con base de datos**
- ✅ Sincronización entre dispositivos
- ✅ Datos persistentes independientemente del navegador
- ❌ Costo de servidor (mínimo ~$5-10/mes si no se usa free tier)
- ❌ Requiere auth (aunque sea simple)
- ❌ Latencia de red en cada operación
- ❌ Un punto más de fallo

**Decisión tomada**: Opción A — frontend-only.

**Criterio desempate**: Sofía nunca solicitó sincronización entre dispositivos (EX-06). Construir un backend para una necesidad no expresada viola el principio YAGNI (You Aren't Gonna Need It).

**Condición para reconsiderar**: Sofía reporta en uso real que el gap entre celular y computadora es un problema concreto que le afecta. En ese momento, la información de su comportamiento real justifica añadir el backend.

---

## Decisión 2: localStorage vs. IndexedDB

**El problema**: ¿Dónde persistir el historial del día en el navegador?

**Opción A: localStorage**
- ✅ API síncrona y simple: `getItem/setItem`
- ✅ Disponible en todos los navegadores sin excepción
- ✅ Suficiente para los volúmenes de datos de este sistema
- ❌ Solo guarda strings (requiere JSON.stringify/parse)
- ❌ Límite de ~5MB
- ❌ API síncrona bloquea el hilo principal (irrelevante para este volumen)

**Opción B: IndexedDB**
- ✅ Soporte para objetos JS nativos
- ✅ Límite de almacenamiento mucho mayor
- ✅ Transacciones ACID
- ✅ Operaciones asíncronas (no bloquea UI)
- ❌ API verbosa y compleja para un caso de uso simple
- ❌ Requiere librerías como `idb` para usarla cómodamente

**Decisión tomada**: Opción A — localStorage.

**Criterio desempate**: la complejidad de IndexedDB no resuelve ningún problema real del sistema. El máximo de datos a guardar es ~2KB. La API síncrona es un "problema" solo si el hilo principal se bloquea por más de 16ms — imposible con 2KB de datos.

**Condición para reconsiderar**: si se implementa DF-02 (historial de días anteriores), el volumen de datos podría justificar IndexedDB.

---

## Decisión 3: Vanilla JS vs. Framework (React/Vue/Svelte)

**El problema**: ¿Con qué tecnología implementar la UI?

**Opción A: Vanilla JavaScript**
- ✅ Cero dependencias, cero configuración
- ✅ Bundles más pequeños
- ✅ Sin lock-in a un framework
- ✅ El código es explícito sobre lo que hace
- ❌ Más código manual para manipulación del DOM
- ❌ Sin reactividad automática (hay que llamar `render()` manualmente)
- ❌ Más difícil de escalar si la UI crece mucho

**Opción B: Framework (React/Vue/Svelte)**
- ✅ Reactividad automática — el DOM se actualiza solo cuando cambia el estado
- ✅ Ecosistema de componentes reutilizables
- ✅ Facilita el testing con React Testing Library
- ❌ Añade dependencias y un build step
- ❌ Overhead conceptual (hooks, ciclo de vida, etc.) para una UI de una pantalla
- ❌ Svelte y Vue tienen curvas de aprendizaje propias

**Decisión tomada**: depende del objetivo educativo del desarrollador.
- Si el objetivo es entender los principios: **Vanilla JS**
- Si el objetivo es practicar React/Vue: cualquiera de los dos es válido

**Criterio desempate**: una SPA de una sola pantalla con ~5 elementos interactivos no justifica un framework por sus propios méritos técnicos. Sin embargo, si Sofía quiere que la app evolucione (DF-01, DF-02), un framework amortigua la complejidad futura.

---

## Decisión 4: Cuándo ejecutar el reinicio diario

**El problema**: ¿Cómo y cuándo se borra el historial del día anterior?

**Opción A: Al abrir la app (comparar fecha guardada con hoy)**
- ✅ Simple de implementar
- ✅ Sin timers corriendo en segundo plano
- ❌ Si la app está abierta a medianoche, el historial no se borra hasta que se recargue

**Opción B: Timer programado (`setTimeout` calculando ms hasta medianoche)**
- ✅ El reinicio ocurre exactamente a medianoche aunque la app esté abierta
- ❌ Los timers del navegador pueden suspenderse cuando la pestaña está en segundo plano
- ❌ Si el usuario deja la pestaña abierta toda la noche, el timer puede no dispararse

**Opción C: Combinación (verificar al abrir + timer de respaldo)**
- ✅ Cubre ambos casos
- ❌ Más código, más casos de test

**Decisión tomada**: Opción A como implementación principal, Opción C para versiones posteriores.

**Criterio desempate**: el caso de "app abierta exactamente a medianoche" es improbable en el patrón de uso de Sofía (registra comidas durante el día, no a medianoche). Opción A resuelve el 99% de los casos con el mínimo de complejidad.

---

## Decisión 5: Identificador de entradas (timestamp vs. UUID)

**El problema**: cada entrada necesita un ID único para poder eliminarla.

**Opción A: `Date.now()` (timestamp en ms)**
- ✅ Zero dependencias
- ✅ Simple de leer y debuggear
- ❌ Colisión si dos entradas se crean en el mismo milisegundo (probabilidad: casi cero)

**Opción B: UUID v4 (`crypto.randomUUID()`)**
- ✅ Colisiones prácticamente imposibles (128 bits de aleatoriedad)
- ✅ Estándar para identificadores únicos distribuidos
- ❌ Requiere verificar soporte del navegador (disponible en todos los navegadores modernos)
- ❌ Más largo (36 caracteres vs. 13 dígitos)

**Decisión tomada**: `crypto.randomUUID()` si se prioriza corrección; `Date.now()` si se prioriza simplicidad.

**Criterio desempate**: para V1 con una usuaria, la colisión de `Date.now()` es teóricamente posible pero prácticamente imposible. Si se quisiera blindar esto sin overhead: `Date.now() + '_' + Math.random().toString(36).slice(2)`.

---

## Decisión 6: Persistir o no la meta calórica en localStorage

**El problema**: ¿La meta calórica debe sobrevivir al reinicio diario?

**Opción A: La meta persiste (se borra solo manualmente)**
- ✅ Sofía no tiene que reconfigurar su meta cada día
- ✅ Comportamiento esperado para una meta personal estable
- ❌ Si Sofía quiere "empezar de cero", necesita borrarla manualmente

**Opción B: La meta se borra con el reinicio diario**
- ❌ Sofía debe reconfigurar su meta cada mañana — claramente incómodo
- ❌ Contradice el comportamiento esperado (OV-04)

**Decisión tomada**: Opción A — la meta persiste. Documentada como RN-06 y RN-19.

**No hay trade-off real aquí** — Opción B no tiene ninguna ventaja que justifique la fricción. Es una decisión obvia una vez que se entiende el contexto.

---

## Lecciones transferibles

1. **La arquitectura debe seguir al contexto, no a la moda**: la decisión de no usar backend no es "hacer las cosas mal" — es hacer exactamente lo que el problema requiere.

2. **El usuario que no pidió algo, probablemente no lo necesita**: EX-06 (sin sincronización entre dispositivos) se mantuvo fuera del scope porque Sofía nunca lo solicitó. Añadirlo sin evidencia de necesidad es diseño especulativo.

3. **La complejidad futura se gestiona con abstracciones hoy, no con implementación anticipada**: el `StorageRepository` no hace más de lo necesario hoy, pero está diseñado para ser reemplazado mañana. Esa es la forma correcta de prepararse para el futuro.

4. **Documentar condiciones para reconsiderar**: una decisión sin condición de revisión es una decisión que envejece sin actualizarse. Cada trade-off aquí tiene una condición explícita de cuándo revisarlo.
