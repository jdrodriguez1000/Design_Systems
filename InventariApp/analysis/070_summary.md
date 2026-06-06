# Resumen Ejecutivo del Diseño — InventariApp

## Visión general

InventariApp es un sistema web de control de inventario y alertas de stock para Distribuidora Andina Ltda., empresa de 18 personas con un único almacén físico. Reemplaza el control en hoja de cálculo que genera quiebres de stock sin aviso, registros desactualizados y descuadres sin trazabilidad.

El desafío técnico central no es la escala (18 usuarios, baja concurrencia), sino la **integridad**: garantizar que el historial de movimientos sea la única fuente de verdad del stock, que las alertas se generen y resuelvan automáticamente con consistencia transaccional, y que los datos mostrados al Gerente General sean confiables o claramente marcados como no confiables.

**Patrón arquitectónico central**: el sistema es un **ledger inmutable** — el stock actual nunca se almacena, siempre se calcula desde el historial append-only de movimientos. Este diseño garantiza auditoría total y consistencia por construcción.

---

## Decisiones arquitectónicas clave

| Decisión | Justificación en una línea |
|----------|---------------------------|
| Monolito por capas | 18 usuarios + lógica transaccional acoplada (movimiento + alerta) → un solo proceso es más simple y correcto |
| PostgreSQL con ACID | El trio movimiento + recálculo de stock + generación de alerta debe ocurrir en una sola transacción atómica |
| Stock calculado desde movimientos (sin campo editable) | RN-01: garantía de consistencia por diseño — imposible que el saldo diverja del historial |
| SELECT FOR UPDATE al registrar movimiento | Evita race condition donde dos movimientos simultáneos crean dos alertas activas (viola RN-13) |
| Observer/Domain Events para evaluación de alertas | AlertEvaluator no acopla su lógica con MovementService — extensible sin modificar el servicio principal |
| Historial append-only (sin DELETE ni UPDATE en movimientos) | Auditoría garantizada por diseño, no por disciplina — las correcciones son contra-movimientos visibles |
| Sin autenticación en v1 | Aceptado por Luis para MVP. Mitigado con restricción a red interna. Pendiente para v2 |
| CAP CP — consistencia sobre disponibilidad | Stock incorrecto genera promesas de venta incumplibles — la inconsistencia temporal tiene costo real |
| Override con nota obligatoria + historial de excepciones | Las salidas sobre stock son casos de negocio reales (2-3/mes) — bloqueo absoluto generaría pérdida de ventas |
| Advertencia "datos no conciliados" antes de KPIs | Luis prefiere un aviso preventivo a ver un número incorrecto con apariencia de certeza |

---

## Mapa de documentos generados

| Documento | Decisión principal |
|-----------|------------------|
| `005_architecture.md` | Monolito por capas con núcleo de ledger — justificación vs. microservicios y BaaS |
| `010_database_model.md` | PostgreSQL, stock calculado (sin campo), índices parciales, SELECT FOR UPDATE |
| `015_design_patterns.md` | Observer para alertas, State Machine para órdenes, Repository, Command |
| `020_data_structures.md` | Log append-only, state machine de transiciones, tabla hash para validaciones |
| `025_communication_protocols.md` | REST + JSON, polling de alertas, sin WebSockets |
| `030_scalability.md` | Vertical mínima para v1; cuellos de botella anticipados si la empresa crece |
| `035_advanced_concepts.md` | CAP CP, ACID completo, SELECT FOR UPDATE para concurrencia de alertas |
| `040_maintainability.md` | AlertEvaluator como módulo aislado, estructura de carpetas, deuda técnica |
| `045_security.md` | Sin auth en v1 (riesgo aceptado), SQL injection prevention, ledger como auditoría |
| `050_testing_strategy.md` | 60% unit / 30% integración / 10% E2E; casos críticos de alertas y stock |
| `055_deployment.md` | Rolling update, Docker Compose, Railway/Render, backups diarios obligatorios |
| `060_tradeoffs.md` | 6 decisiones difíciles con opciones, criterios de desempate y condiciones de revisión |
| `065_tech_stack.md` | FastAPI + React + PostgreSQL + Railway con justificación por capa |
| `070_summary.md` | Este documento |

---

## Roadmap de implementación

### Fase 1 — Núcleo del ledger (Sprint 1-2)
1. Setup: Docker Compose (backend FastAPI + PostgreSQL) + migraciones iniciales con Alembic
2. Catálogo de productos: CRUD de productos, categorías y proveedores
3. Registro de movimientos: entradas y salidas con validación de fecha no futura
4. Cálculo de stock: `StockCalculator` con aggregation query + tests unitarios
5. Generación y resolución automática de alertas: `AlertEvaluator` con tests de todos los casos del `TestAlertEvaluator`
6. Tests de integración: transaccionalidad movimiento + alerta con PostgreSQL real

### Fase 2 — Contra-movimientos, override y alertas en UI (Sprint 3)
1. Contra-movimientos con nota obligatoria y referencia al movimiento de origen
2. Override (despacho en rojo): bloqueo + opción de forzar con nota + flag excepción
3. Pantalla principal de Andrés: alertas activas visibles sin navegar
4. Vista de detalle de alerta con contexto de 30 días (IS-05)
5. Historial de movimientos por producto con indicador visual de excepciones

### Fase 3 — Módulo Diana: órdenes y autoservicio (Sprint 4)
1. Configuración de stock mínimo en autoservicio (IS-07) con advertencia cuando = 0
2. Órdenes de reposición: ciclo de vida completo (Borrador → Confirmada → Recibida)
3. Movimiento de entrada automático al marcar Recibida (RN-18)
4. Historial de excepciones: vista dedicada en menú principal (DD-03)

### Fase 4 — Panel Luis y observabilidad (Sprint 5)
1. KPIs ejecutivos: alertas activas + excepciones del mes (IS-10)
2. Advertencia de datos no conciliados con criterios de disparo (IS-11, DD-02)
3. Tabla de productos con filtrado por categoría y estado (IS-13)
4. Logging estructurado JSON + integración Sentry
5. Pipeline CI/CD completo con GitHub Actions + deploy a Railway/Render
6. Backups automáticos diarios configurados

---

## Riesgos técnicos principales

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|-----------|
| **Alerta no generada por bug en evaluación** | Media | Crítico — stock agotado sin aviso | Test suite exhaustiva de `AlertEvaluator` con todos los casos (RN-02, RN-13, RN-14, RN-16); tests de integración transaccionales |
| **Race condition en evaluación de alertas** | Baja (principalmente un operador) | Alto — alertas duplicadas | `SELECT FOR UPDATE` en el producto al registrar cualquier movimiento |
| **Stock incorrecto por falla de transacción a mitad** | Baja (ACID) | Crítico | Transacción atómica garantizada por PostgreSQL; rollback automático en cualquier fallo |
| **Pérdida del ledger por corrupción de BD sin backups** | Baja (si se configura) | Muy alto — pérdida total del historial de operaciones | Backups diarios automáticos desde el día 1 — esta es la acción de infraestructura más importante |
| **Sin trazabilidad de "quién hizo qué" en v1** | Alta (sin auth) | Bajo-Medio — dificulta resolución de disputas | Mitigado por el ledger inmutable (qué y cuándo es siempre rastreable); pendiente para v2 |

---

## Métricas de éxito en producción

| Métrica | Objetivo | Cómo medirlo |
|---------|---------|-------------|
| **Cero quiebres de stock sin alerta previa** | 0 incidentes | Historial de alertas vs. historial de movimientos: toda salida a stock = 0 debe tener alerta activa |
| **Latencia de registro de movimiento** | P95 < 200ms | Logs estructurados con `duration_ms` |
| **Cobertura de alertas** | 100% de productos con stock ≤ mínimo tienen alerta activa | Query de verificación mensual |
| **Reducción de tiempo en conciliación** | < 15 minutos por descuadre (vs. medio día actual) | Feedback de Andrés en uso real |
| **Adopción del sistema** | < 1 uso de hoja de cálculo por semana al mes 2 | Feedback de Ana (Diana) y el equipo |
| **Integridad del historial** | 0 movimientos perdidos en caídas de sistema | Verificación post-caída: contra-movimientos retroactivos completan el historial |
