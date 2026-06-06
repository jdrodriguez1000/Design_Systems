# Mantenibilidad — MeetingApp

## La complejidad de negocio exige una estructura cuidadosa

Este sistema tiene significativamente más lógica de negocio que Contador_Calorias: prioridades, desplazamientos, roles, retroactivas, health-check. Sin una estructura clara, la lógica de conflictos se dispersa por todo el código y se vuelve imposible de modificar sin riesgo de regresiones.

---

## Principios SOLID aplicados

### S — Responsabilidad Única (crítico)

Cada clase/módulo tiene una razón para cambiar:
- `ConflictResolver`: cambia si cambian las reglas de prioridad
- `NotificationService`: cambia si se añaden nuevos tipos de notificación o canales (DF-02)
- `PriorityEngine`: cambia si se añaden nuevos tipos de reunión o criterios de prioridad
- `RoomAvailabilityService`: cambia si cambia la definición de "disponible" (ej: tiempo de limpieza entre reservas)

**Señal de alerta**: si añadir un nuevo tipo de notificación requiere modificar `ReservationService`, hay una violación de SRP.

### O — Abierto/Cerrado (crítico para extensibilidad)

Las estrategias de resolución de conflictos (`DisplacementStrategy`, `RejectionStrategy`) permiten añadir nuevos comportamientos sin modificar los existentes. Si el negocio decide añadir un tercer nivel de prioridad ("ultra-prioritaria" para el CEO), solo se añade una nueva estrategia.

### D — Inversión de Dependencias (crítico para testing)

`ReservationService` depende de abstracciones (`ReservationRepository`, `NotificationService`), no de implementaciones concretas. Esto permite inyectar mocks en tests.

---

## Separación de concerns

**Estructura de módulos recomendada**:

```
src/
├── api/
│   ├── routes/           # Solo define endpoints y llama a services
│   ├── middleware/        # Auth, RBAC, error handling
│   └── serializers/       # Request/response schemas
├── domain/
│   ├── reservation/
│   │   ├── service.py     # Lógica orquestadora
│   │   ├── conflict.py    # ConflictResolver + estrategias
│   │   └── priority.py    # PriorityEngine
│   ├── room/
│   │   └── availability.py
│   └── notification/
│       └── service.py
├── infrastructure/
│   ├── db/
│   │   ├── repositories/  # Implementaciones concretas
│   │   └── migrations/    # Esquema de BD versionado
│   └── auth/
│       └── jwt.py
└── tests/
    ├── unit/              # Lógica de dominio sin BD
    └── integration/       # Con BD real (Docker en CI)
```

**Regla de dependencia**: los módulos de `domain/` no importan nada de `infrastructure/` ni de `api/`. Solo conocen abstracciones (interfaces). Los módulos de `infrastructure/` implementan las abstracciones definidas en `domain/`.

---

## Modularidad

**¿Qué tan fácil es reemplazar o modificar cada parte?**

| Componente | Facilidad de reemplazo | Condición |
|-----------|----------------------|-----------|
| Base de datos (PostgreSQL → MySQL) | Media | Solo cambia `infrastructure/db/` si el repository pattern está bien implementado |
| Estrategia de notificaciones (in-app → correo) | Alta | Añadir `EmailNotificationService` que implementa la misma interfaz |
| Estrategia de resolución de conflictos | Alta | Añadir nueva `Strategy` sin tocar el código existente |
| Lógica de autenticación (JWT → sesiones) | Media | Solo cambia `infrastructure/auth/` y el middleware |
| Frontend (React → Vue) | Alta | La API REST es independiente del frontend |

---

## Deuda técnica anticipada

| Atajo | Deuda que genera | Cuándo mitigar |
|-------|-----------------|---------------|
| Hardcodear horario laboral (8-19h) en el CHECK constraint | Si la empresa cambia el horario, es un ALTER TABLE en producción | Configuración externa antes de V2 |
| Número de salas fijo en 3 (RN-09) | Añadir una sala requiere revisar queries que asumen 3 resultados | Parametrizar desde V1: `SELECT * FROM salas` en lugar de `LIMIT 3` |
| Notificaciones con polling simple | Si se necesita tiempo real (DF-02), hay que añadir SSE/WebSockets y refactorizar el cliente | Diseñar el endpoint de notificaciones como si fuera a ser replaced por SSE — mismos eventos, diferente canal |
| Sin soft-delete de usuarios | Desactivar un usuario no elimina sus reservas futuras — se quedan huérfanas | Añadir `activo` flag y filtrar en queries de disponibilidad |

---

## Estrategia de versionado de la API

**Versionado en URL**: `/api/v1/reservas`

Cuando cambie un contrato de API que rompa clientes existentes:
1. Crear `/api/v2/reservas` con el nuevo contrato
2. Mantener `/api/v1/reservas` durante un período de transición
3. Deprecar v1 solo cuando todos los clientes hayan migrado

Para una app con un único frontend interno, la migración es controlada. Pero documentar el versionado desde V1 evita que el primer cambio rompiente sea un caos.

---

## Code smells específicos a vigilar

| Code Smell | Señal concreta | Solución |
|-----------|---------------|---------|
| **Lógica de negocio en endpoints** | `if user.rol == 'gerente':` dentro de un controlador | Mover a `PriorityEngine` o decorator RBAC |
| **Queries SQL en la capa de servicio** | `db.execute("SELECT...")` dentro de `ReservationService` | El servicio habla con el repositorio, nunca con la DB directamente |
| **Magic strings** | `if estado == 'activa':` disperso en 10 lugares | Enum o constante `ReservationStatus.ACTIVE` |
| **God Service** | `ReservationService` con 500+ líneas haciendo todo | Extraer `ConflictResolver`, `DisplacementExecutor`, `AlternativeRoomFinder` |
| **Test sin aislamiento** | Tests de conflicto que comparten datos en BD | Cada test resetea su estado; no compartir fixtures entre tests de conflicto |
