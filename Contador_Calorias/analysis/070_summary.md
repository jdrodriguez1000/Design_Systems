# Resumen Ejecutivo del Diseño — Contador de Calorías

## Visión general

"Mi Contador de Calorías" es una Single-Page Application (SPA) de frontend puro diseñada para un único usuario (Sofía Martínez). Permite registrar entradas de comida con nombre y calorías, consultar el total consumido del día, y compararlo con una meta calórica personal editable. Los datos persisten en `localStorage` del navegador y se reinician automáticamente a medianoche. No hay servidor, no hay backend, no hay autenticación.

**Objetivo de diseño central**: máxima simplicidad que resuelve el problema real. Cero complejidad accidental.

---

## Decisiones arquitectónicas clave

| Decisión | Justificación en una línea |
|----------|---------------------------|
| SPA frontend-only, sin backend | Un solo usuario sin necesidad de sync entre dispositivos hace innecesario cualquier servidor |
| localStorage como almacenamiento | Persiste entre sesiones del navegador con API nativa, sin dependencias externas |
| Reinicio diario client-side al cargar | Simple y suficiente; el caso de app abierta exactamente a medianoche es de baja probabilidad |
| Sin framework para V1 | Vanilla JS + Vite cubre la complejidad de una pantalla sin overhead de un framework |
| Entidades calculadas, no almacenadas | Total consumido, saldo y exceso se derivan del array de entradas — evita inconsistencias |
| IDs por timestamp | Cero dependencias para identificadores únicos en un contexto de usuario único |

---

## Mapa de documentos generados

| Documento | Decisión principal que contiene |
|-----------|--------------------------------|
| `005_architecture.md` | Por qué SPA frontend-only y no añadir backend |
| `010_database_model.md` | Por qué localStorage, esquema de claves y evolución futura |
| `015_design_patterns.md` | Repository Pattern, Strategy para estados de UI, Observer para reactividad |
| `020_data_structures.md` | Array para entradas, enteros para calorías, timestamp como ID |
| `025_communication_protocols.md` | Sin red en V1; análisis de REST vs GraphQL para DF-04 |
| `030_scalability.md` | La escalabilidad no es el reto actual; análisis si el sistema evoluciona |
| `035_advanced_concepts.md` | CAP theorem (AP si hay backend), ACID vs BASE en localStorage |
| `040_maintainability.md` | SOLID aplicado, separación de módulos, deuda técnica anticipada |
| `045_security.md` | XSS como única vulnerabilidad relevante; mitigado con `textContent` |
| `050_testing_strategy.md` | 70% unit (lógica pura), 20% integración (localStorage), 10% E2E |
| `055_deployment.md` | GitHub Pages / Netlify, pipeline mínimo de CI/CD |
| `060_tradeoffs.md` | Las 6 decisiones difíciles del diseño con sus trade-offs y condiciones de revisión |
| `065_tech_stack.md` | Vite + Vanilla JS + Vitest; stack completo por capa |
| `070_summary.md` | Este documento |

---

## Roadmap de implementación

### Fase 1 — Núcleo funcional (Sprint 1)
1. Setup: `index.html` + `src/` con módulos separados
2. `storage.js` — StorageRepository con todas las operaciones de localStorage
3. `calculator.js` — funciones puras: `sumEntries`, `calculateBalance`
4. `validator.js` — `validateCalories` con los 4 tipos de error
5. Tests unitarios de los 3 módulos anteriores (cubrir todos los ACP de SE-01 a SE-10)
6. `renderer.js` — actualizar DOM desde el estado
7. `app.js` — orquestador: conectar todos los módulos
8. Verificación manual de los 8 escenarios de camino feliz (SC-01 a SC-08)

### Fase 2 — Pulido de UX (Sprint 2)
1. Diseño visual: colores de exceso (rojo/naranja), layout responsive para celular
2. Lógica de reinicio diario: verificación al cargar + timer opcional a medianoche
3. Tests E2E de los flujos principales
4. Deploy a GitHub Pages / Netlify

### Fase 3 — Diferidos (según demanda de Sofía)
- DF-01: Edición directa de entradas
- DF-02: Historial de días anteriores (requiere rediseño del modelo de datos)
- DF-03: Catálogo de alimentos frecuentes
- DF-04: Sincronización entre dispositivos (requiere backend completo)

---

## Riesgos técnicos principales

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|-----------|
| **Pérdida de datos por limpieza de caché del navegador** | Media | Alto (pierde historial del día) | Documentar a Sofía que esto puede ocurrir; en V2, añadir exportación a JSON |
| **Reinicio diario no ocurre si la app está abierta a medianoche** | Baja | Medio (datos del día anterior visibles por unas horas) | Añadir timer de respaldo en V2 |
| **XSS si se usa `innerHTML` accidentalmente** | Baja | Medio | Code review; linter rule para prohibir `innerHTML` |
| **Colisión de IDs con `Date.now()`** | Muy baja | Bajo (duplicado que no se puede eliminar individualmente) | Documentado; aceptable para V1 |
| **localStorage lleno** | Muy baja | Alto (no puede guardar nuevas entradas) | Manejar el error de `setItem` y mostrar mensaje claro |

---

## Métricas de éxito en producción

| Métrica | Valor objetivo | Cómo medirlo |
|---------|---------------|-------------|
| **Sofía puede registrar una comida en < 10 segundos** | Sí | Prueba manual en celular |
| **El total consumido siempre es correcto** | 100% de precisión | Tests unitarios de `sumEntries` |
| **Los datos persisten tras cerrar el navegador** | 100% de casos | Test de integración de localStorage |
| **El reinicio diario borra entradas y preserva la meta** | 100% de casos | Test unitario de `checkReset` |
| **La app carga en < 2 segundos en 3G** | < 2s | Bundle size < 50KB; verificar con DevTools |
| **Cero entradas inválidas guardadas** | 0 | Tests unitarios de `validateCalories` |
