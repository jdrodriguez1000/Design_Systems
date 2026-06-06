# Patrones de Diseño — MeetingApp

## Los patrones seleccionados resuelven problemas reales de este dominio

*Como se documentó en Contador_Calorias: los patrones son soluciones a problemas recurrentes. Si el problema no existe, el patrón no se necesita. Aquí hay problemas reales y complejos que justifican patterns más sofisticados.*

---

## 1. Repository Pattern (Arquitectónico — Estructural)

### El problema
La lógica de negocio (detectar conflictos, ejecutar desplazamientos, clasificar prioridades) no debe saber si los datos vienen de PostgreSQL, de un mock en tests, o en el futuro de otra fuente.

*Como se aprendió en Contador_Calorias: el Repository Pattern permite cambiar el mecanismo de persistencia sin tocar la lógica de negocio. Aquí aplica con mayor fuerza — la lógica de conflictos es compleja y tener que reescribirla si se cambia la BD sería un riesgo innecesario.*

### Aplicación

```python
# Contrato del repositorio — la lógica de negocio solo conoce esta interfaz
class ReservationRepository:
    def find_active_overlapping(self, sala_id, inicio, fin) -> list[Reserva]: ...
    def find_active_by_user_and_interval(self, user_id, inicio, fin) -> list[Reserva]: ...
    def save(self, reserva: Reserva) -> Reserva: ...
    def cancel(self, reserva_id: UUID) -> None: ...
    def lock_room_for_update(self, sala_id: UUID) -> None: ...  # SELECT FOR UPDATE

# La lógica de negocio solo habla con la interfaz
class ReservationService:
    def __init__(self, repo: ReservationRepository, notif: NotificationService):
        self.repo = repo
        self.notif = notif

    def create_reservation(self, data: ReservationRequest) -> Reserva:
        with transaction():
            self.repo.lock_room_for_update(data.sala_id)  # evita race conditions
            conflicts = self.repo.find_active_overlapping(data.sala_id, data.inicio, data.fin)
            # ... lógica de resolución
```

**Ventaja concreta en tests**: `MockReservationRepository` con datos en memoria permite testear toda la lógica de conflictos sin una BD real.

---

## 2. Strategy Pattern — ConflictResolutionStrategy (Comportamiento)

### El problema
Hay dos tipos de conflicto con resolución diferente:
1. **Diferente prioridad**: la de mayor prioridad gana → desplazamiento automático
2. **Igual prioridad máxima**: gana la más antigua → rechazo de la segunda

Sin este patrón, el código de resolución sería un bloque `if/else` que crecerá con el tiempo (¿qué pasa si se añade un tercer nivel de prioridad?).

*Del historial de Contador_Calorias: el Strategy Pattern encapsula cada comportamiento como una función pura con un contrato claro, extensible sin modificar las estrategias existentes (Open/Closed).*

### Aplicación

```python
class ConflictResolutionStrategy:
    def resolve(self, incoming: Reserva, existing: Reserva) -> ConflictResult: ...

class DisplacementStrategy(ConflictResolutionStrategy):
    """Prioridad entrante > Prioridad existente → desplazar"""
    def resolve(self, incoming, existing) -> ConflictResult:
        return ConflictResult(action='displace', victim=existing)

class RejectionStrategy(ConflictResolutionStrategy):
    """Igual prioridad → la más antigua gana, la nueva se rechaza"""
    def resolve(self, incoming, existing) -> ConflictResult:
        winner = existing if existing.timestamp_creacion < incoming.timestamp_creacion else incoming
        loser = incoming if winner == existing else existing
        return ConflictResult(action='reject', victim=loser)

class ConflictResolver:
    def select_strategy(self, incoming: Reserva, existing: Reserva) -> ConflictResolutionStrategy:
        if incoming.tipo_prioridad == 'prioritaria' and existing.tipo_prioridad == 'estandar':
            return DisplacementStrategy()
        if incoming.tipo_prioridad == existing.tipo_prioridad == 'prioritaria':
            return RejectionStrategy()
        return NoConflictStrategy()  # entrante es estándar, existente es prioritaria → rechazar silenciosamente
```

---

## 3. Observer / Domain Events Pattern (Comportamiento)

### El problema
Cuando ocurre un desplazamiento, múltiples cosas deben suceder: cancelar la reserva, crear la nueva, registrar el desplazamiento, y **notificar al afectado**. Si el código de notificación está mezclado con el código de reserva, el acoplamiento hace que añadir un nuevo tipo de notificación (ej: DF-02, correo) requiera modificar `ReservationService`.

### Aplicación

```python
# Evento de dominio
@dataclass
class ReservationDisplacedEvent:
    displaced_reservation: Reserva
    incoming_reservation: Reserva
    alternative_room: Optional[Sala]

# ReservationService publica el evento — no sabe qué pasará con él
class ReservationService:
    def create_reservation(self, data):
        with transaction():
            # ... lógica de conflicto
            if conflict.action == 'displace':
                self.repo.cancel(conflict.victim.id)
                new_reservation = self.repo.save(incoming)
                alternative = self.find_alternative_room(...)
                self.event_bus.publish(ReservationDisplacedEvent(
                    displaced_reservation=conflict.victim,
                    incoming_reservation=new_reservation,
                    alternative_room=alternative
                ))
                return new_reservation

# NotificationService escucha el evento — no sabe nada de reservas
class NotificationService:
    def on_reservation_displaced(self, event: ReservationDisplacedEvent):
        message = self.build_displacement_message(event)
        self.repo.save(Notificacion(
            destinatario_id=event.displaced_reservation.usuario_id,
            tipo='cancelacion_desplazamiento',
            mensaje=message,
            sala_alternativa_id=event.alternative_room.id if event.alternative_room else None
        ))
```

**Ventaja**: cuando en el futuro se implemente DF-02 (notificaciones por correo), solo se añade un nuevo listener al evento — sin tocar `ReservationService`.

---

## 4. Command Pattern (Comportamiento)

### El problema
Las operaciones de reserva son complejas y tienen efectos secundarios (notificaciones, desplazamientos). Modelarlas como Commands hace el código más testeable y permite añadir funcionalidades como auditoría o undo.

```python
# Cada operación es un objeto con toda la información necesaria
@dataclass
class CreateReservationCommand:
    usuario_id: UUID
    sala_id: UUID
    fecha_inicio: datetime
    fecha_fin: datetime
    tipo_reunion: Optional[str]
    is_retroactive: bool = False

# El handler ejecuta el comando
class CreateReservationHandler:
    def handle(self, cmd: CreateReservationCommand) -> Reserva:
        # Determinar prioridad automáticamente
        prioridad = 'prioritaria' if cmd.tipo_reunion in ['cliente_externo', 'directorio'] else 'estandar'
        # ... resto de la lógica
```

---

## 5. RBAC — Role-Based Access Control (Arquitectónico)

### El problema
Tres acciones son exclusivas de roles específicos: marcar urgente (solo gerente), crear retroactivas (solo administradora), crear usuarios (solo administradora). Sin un sistema RBAC explícito, estas verificaciones quedan dispersas como condiciones `if` en cada endpoint.

```python
# Permisos declarados centralmente — no dispersos en endpoints
PERMISSIONS = {
    'empleado_general': {'reservar', 'consultar', 'cancelar_propia'},
    'gerente':          {'reservar', 'consultar', 'cancelar_propia', 'marcar_urgente'},
    'administradora':   {'reservar', 'consultar', 'cancelar_propia', 'crear_usuarios',
                         'crear_retroactiva', 'cancelar_cualquiera'}
}

def require_permission(permission: str):
    """Decorador de autorización — aplicado en cada endpoint"""
    def decorator(handler):
        def wrapper(request, *args, **kwargs):
            user_role = request.user.rol
            if permission not in PERMISSIONS.get(user_role, set()):
                raise ForbiddenError(f"Rol {user_role} no tiene permiso: {permission}")
            return handler(request, *args, **kwargs)
        return wrapper
    return decorator

# Uso en endpoints
@require_permission('marcar_urgente')
def mark_as_urgent(request, reservation_id):
    # Solo llega aquí si el usuario es gerente
    ...
```

---

## Patrones que NO se aplican aquí

| Patrón | Por qué no |
|--------|-----------|
| **CQRS** | Separa lecturas y escrituras en modelos distintos — útil cuando los patrones de lectura y escritura son muy diferentes (ej: writes complejos, reads simples). Para este sistema, el mismo modelo sirve para ambos |
| **Event Sourcing** | Reconstruir el estado desde eventos tiene valor para auditoría completa. Aquí la tabla `desplazamientos` ya provee la trazabilidad necesaria sin la complejidad del event sourcing |
| **Saga** | Para orquestar transacciones distribuidas entre microservicios. En un monolito con una sola BD, una transacción ACID reemplaza toda la complejidad de Saga |
