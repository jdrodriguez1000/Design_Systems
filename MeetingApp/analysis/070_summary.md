# Resumen Ejecutivo del Diseño — MeetingApp

## Visión general

MeetingApp es un sistema web de reserva de salas de reunión para una empresa de 25 personas con 3 salas disponibles. Reemplaza la coordinación manual por WhatsApp con un sistema autónomo que gestiona disponibilidad, roles, y conflictos de prioridad automáticamente. El desafío técnico central es la **concurrencia con consistencia fuerte**: garantizar que ninguna sala tenga dos reservas activas en el mismo horario, incluso bajo condiciones de carrera.

**Objetivo de diseño central**: consistencia fuerte en la lógica de conflictos, simplicidad en la arquitectura, y extensibilidad incremental (las capacidades diferidas DF-01 a DF-05 se pueden añadir sin reescribir el núcleo).

---

## Decisiones arquitectónicas clave

| Decisión | Justificación en una línea |
|----------|---------------------------|
| Monolito por capas (no microservicios) | 25 usuarios + lógica transaccional compleja → un solo proceso es más simple y más correcto |
| PostgreSQL con ACID | La lógica de desplazamiento requiere transacciones atómicas; el solapamiento de intervalos es SQL nativo |
| SELECT FOR UPDATE (locking pesimista) | Garantía absoluta de no doble-booking bajo concurrencia; baja latencia aceptable para este volumen |
| Notificaciones in-app en la misma transacción | Consistencia: no hay notificaciones de operaciones que no completaron |
| JWT con access token corto (15-30 min) | Stateless, sin Redis extra; cambios de rol con delay de minutos es aceptable para este contexto |
| Roles RBAC verificados server-side siempre | La UI que oculta botones es UX, no seguridad |
| Alerta retroactiva informativa (no bloqueante) | La administradora tiene el contexto completo del fallback; el sistema no puede resolver la ambigüedad mejor que ella |
| CP en el teorema CAP | Una inconsistencia en reservas es visible y costosa; preferir consistencia sobre disponibilidad |

---

## Mapa de documentos generados

| Documento | Decisión principal |
|-----------|------------------|
| `005_architecture.md` | Monolito por capas — justificación vs. microservicios y serverless |
| `010_database_model.md` | PostgreSQL con ACID, esquema completo, SELECT FOR UPDATE |
| `015_design_patterns.md` | Repository, Strategy (conflictos), Observer (eventos), Command, RBAC |
| `020_data_structures.md` | Algoritmo de solapamiento de intervalos, UUID, cola de notificaciones |
| `025_communication_protocols.md` | REST con JSON, polling vs. SSE para notificaciones en tiempo real |
| `030_scalability.md` | Vertical simple para V1; análisis de cuellos de botella si crece |
| `035_advanced_concepts.md` | CAP→CP, ACID completo, locking pesimista, retry policy para deadlocks |
| `040_maintainability.md` | SOLID aplicado, separación de módulos, deuda técnica anticipada |
| `045_security.md` | JWT + RBAC server-side, bcrypt, OWASP Top 5 relevantes, auditoría |
| `050_testing_strategy.md` | 60% unit (lógica pura) / 30% integración (BD real) / 10% E2E |
| `055_deployment.md` | Rolling update, Docker Compose, Railway/Render, health-check = notificación restablecimiento |
| `060_tradeoffs.md` | Las 6 decisiones difíciles con opciones, criterio de desempate y condición de revisión |
| `065_tech_stack.md` | FastAPI + React + PostgreSQL + Railway por capa con justificación |
| `070_summary.md` | Este documento |

---

## Roadmap de implementación

### Fase 1 — Núcleo de reservas (Sprint 1-2)
1. Setup: Docker Compose (backend + PostgreSQL) + migraciones iniciales
2. Autenticación: JWT login, middleware de auth, RBAC decorator
3. Gestión de usuarios: `POST /usuarios` (solo admin)
4. Consulta de disponibilidad: `GET /salas/disponibilidad` con query de solapamiento
5. Crear reserva estándar: `POST /reservas` con `SELECT FOR UPDATE`
6. Tests unitarios de `ConflictResolver` y overlap detection
7. Tests de integración con BD real (incluyendo concurrencia)

### Fase 2 — Sistema de prioridades y desplazamiento (Sprint 3)
1. Motor de prioridades: clasificación automática por `tipo_reunion`
2. Desplazamiento automático: `DisplacementStrategy` + transacción atómica
3. Empate de prioridad: `RejectionStrategy` con timestamp_creacion
4. Marcado manual urgente: `PATCH /reservas/:id/urgente` (solo gerente)
5. Notificaciones in-app: tabla + endpoint `GET /notificaciones`
6. Tests de todos los escenarios de desplazamiento y empate

### Fase 3 — Operaciones de administración y resiliencia (Sprint 4)
1. Reservas retroactivas con alerta informativa
2. Health-check al arranque → notificación in-app a administradora
3. Logging estructurado JSON + integración Sentry
4. Deploy a producción en Railway/Render
5. Pipeline CI/CD completo (GitHub Actions)

### Fase 4 — Capacidades diferidas (según demanda)
- DF-01: Filtrado de salas por capacidad/equipamiento
- DF-02: Notificaciones por correo/push (Transactional Outbox pattern)
- DF-03: Integración con calendarios externos
- DF-04: Reportes de ocupación

---

## Riesgos técnicos principales

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|-----------|
| **Race condition en reservas concurrentes** | Media (sistema en producción) | Crítico — doble booking | `SELECT FOR UPDATE` implementado desde V1 |
| **Deadlock entre transacciones** | Baja | Alto — transacciones bloqueadas | Retry policy con backoff exponencial (máx 3 reintentos) |
| **JWT robado → acceso no autorizado** | Baja | Alto | Access token de 15-30 min; refresh token en httpOnly cookie |
| **Horario laboral hardcodeado (8-19h)** | Media | Medio — reservas rechazadas erróneamente si cambia el horario | Configuración externa antes de V2 |
| **Pérdida de datos por falta de backups** | Baja (si se configura) | Muy alto | Backups automáticos diarios de PostgreSQL desde el día 1 |

---

## Métricas de éxito en producción

| Métrica | Objetivo | Cómo medirlo |
|---------|---------|-------------|
| **Zero doble-bookings** | 0 incidentes | Log de `SELECT FOR UPDATE` + alertas de conflicto detectado |
| **Latencia de reserva** | P95 < 200ms | Logs estructurados con `duration_ms` |
| **Desplazamientos ejecutados correctamente** | 100% de los casos | Tests + log de desplazamientos |
| **Notificaciones entregadas** | 100% de reservas confirmadas/canceladas | `COUNT(notificaciones)` vs `COUNT(reservas)` afectadas |
| **Disponibilidad** | > 99% (< 7h downtime/mes) | Uptime monitoring (UptimeRobot gratuito) |
| **Adopción del sistema** | < 1 coordinación por WhatsApp/día al mes 2 | Feedback de Ana y el equipo |
