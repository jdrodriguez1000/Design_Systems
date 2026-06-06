# Patrones de Diseño — InventariApp

## Los patrones están al servicio de tres problemas reales del dominio

1. **El registro de un movimiento dispara múltiples efectos secundarios** (cálculo de stock, evaluación de alerta, registro de auditoría). El código que hace todo esto en un bloque lineal se vuelve frágil al agregar nuevos efectos.
2. **El ciclo de vida de una orden de reposición tiene transiciones estrictas y no reversibles** (Borrador → Confirmada → Recibida). Un estado machine explícito previene transiciones inválidas.
3. **La lógica de negocio (alertas, stock, validaciones) no debe acoplarse al mecanismo de persistencia**. El Repository Pattern permite testear la lógica sin base de datos.

---

## 1. Observer / Domain Events (Comportamiento) — el más crítico

### El problema

Cuando Andrés registra un movimiento, deben ocurrir en cadena:
1. El movimiento se inserta en el historial
2. El stock del producto se recalcula
3. Si el nuevo stock ≤ mínimo y no hay alerta activa → generar alerta
4. Si el nuevo stock > mínimo y hay alerta activa → resolver alerta
5. Si el movimiento llegó a estado "Recibida" desde una orden → cerrar la orden

Sin un mecanismo explícito, toda esta lógica termina en `MovementService.register()` como una serie de llamadas directas. Si en el futuro se añade envío de email (futura capacidad diferida), hay que modificar `MovementService` — viola el principio de responsabilidad única.

### Aplicación

```python
# Evento de dominio publicado tras insertar el movimiento
@dataclass
class MovementRegisteredEvent:
    movement_id: UUID
    producto_id: UUID
    tipo_impacto: str  # 'SUMA' o 'RESTA'
    cantidad: int
    nuevo_stock: int   # calculado antes de publicar el evento

# MovementService publica el evento y NO sabe qué pasará después
class MovementService:
    def register(self, cmd: RegisterMovementCommand) -> Movement:
        with transaction():
            movement = self.repo.insert(cmd)
            nuevo_stock = self.stock_calculator.calculate(cmd.producto_id)
            self.event_bus.publish(MovementRegisteredEvent(
                movement_id=movement.id,
                producto_id=cmd.producto_id,
                tipo_impacto=cmd.tipo_impacto,
                cantidad=cmd.cantidad,
                nuevo_stock=nuevo_stock
            ))
            return movement

# AlertEvaluator escucha el evento — no sabe nada de movimientos
class AlertEvaluator:
    def on_movement_registered(self, event: MovementRegisteredEvent):
        producto = self.product_repo.get(event.producto_id)
        alerta_activa = self.alert_repo.get_active(event.producto_id)

        if event.nuevo_stock <= producto.stock_minimo and producto.stock_minimo > 0:
            if not alerta_activa:
                self.alert_repo.create(AlertaStockBajo(
                    producto_id=event.producto_id,
                    stock_al_disparo=event.nuevo_stock
                ))
        elif event.nuevo_stock > producto.stock_minimo and alerta_activa:
            self.alert_repo.resolve(alerta_activa.id)
```

*Como se aprendió en MeetingApp: Observer/Domain Events permite añadir nuevos efectos secundarios (futura notificación por email) sin modificar el servicio principal.*

---

## 2. State Machine — ciclo de vida de OrdenReposicion (Comportamiento)

### El problema

Una orden solo puede transitar Borrador → Confirmada → Recibida. No es reversible. Cuando pasa a Recibida, el sistema debe crear automáticamente un movimiento de entrada. Sin un state machine explícito, estas reglas quedan dispersas como condiciones `if` en el servicio.

### Aplicación

```python
# Transiciones válidas declaradas centralmente
VALID_TRANSITIONS = {
    'borrador':    {'confirmada'},
    'confirmada':  {'recibida'},
    'recibida':    set()  # estado final
}

class OrderService:
    def transition(self, order: Orden, new_state: str) -> Orden:
        if new_state not in VALID_TRANSITIONS[order.estado]:
            raise InvalidTransitionError(
                f"No se puede pasar de '{order.estado}' a '{new_state}'"
            )
        if new_state == 'confirmada':
            self._validate_cantidad_ajustada(order)  # RN-17
        if new_state == 'recibida':
            self._create_entry_movement(order)        # RN-18
        return self.repo.update_state(order.id, new_state)

    def _create_entry_movement(self, order: Orden):
        # Al marcar Recibida, crea el movimiento de entrada automáticamente
        self.movement_service.register(RegisterMovementCommand(
            producto_id=order.producto_id,
            tipo='entrada',
            subtipo='compra',
            tipo_impacto='SUMA',
            cantidad=order.cantidad_ajustada,
            nota=f"Entrada automática desde orden de reposición {order.id}"
        ))
```

---

## 3. Repository Pattern (Arquitectónico)

### El problema

La lógica de negocio compleja (evaluación de alertas, cálculo de stock) no debe saber si los datos vienen de PostgreSQL, un mock en memoria, o en el futuro de otra fuente.

*Como se aprendió en Contador_Calorias: el Repository Pattern permite cambiar el mecanismo de persistencia sin tocar la lógica de negocio. Aquí aplica con fuerza porque la lógica de alertas es la parte más crítica del sistema y necesita ser testeada de forma aislada.*

```python
# Contrato del repositorio de movimientos — append-only explícito
class MovementRepository:
    def insert(self, cmd: RegisterMovementCommand) -> Movement: ...
    def get_by_product(self, producto_id: UUID) -> list[Movement]: ...
    def get_exceptions(self, since: datetime = None) -> list[Movement]: ...
    def get_retroactive_recent(self, hours: int = 24) -> list[Movement]: ...
    # NOTA: no existe update() ni delete() — diseño append-only por contrato

# Contrato del repositorio de alertas
class AlertRepository:
    def get_active(self, producto_id: UUID) -> Optional[AlertaStockBajo]: ...
    def get_all_active(self) -> list[AlertaStockBajo]: ...
    def create(self, alert: AlertaStockBajo) -> AlertaStockBajo: ...
    def resolve(self, alert_id: UUID) -> None: ...
```

---

## 4. Command Pattern para registro de movimientos (Comportamiento)

### El problema

Un movimiento de entrada, una salida estándar, un override, y un contra-movimiento comparten el mismo mecanismo de inserción pero tienen reglas de validación diferentes. Modelarlos como Commands hace el código testeable y clarifica qué campos son obligatorios en cada caso.

```python
@dataclass
class RegisterMovementCommand:
    producto_id: UUID
    tipo: str              # 'entrada' | 'salida' | 'contra-movimiento'
    subtipo: Optional[str]
    tipo_impacto: str      # 'SUMA' | 'RESTA'
    cantidad: int
    fecha_hora_evento: datetime
    referencia: Optional[str] = None
    nota: Optional[str] = None
    flag_excepcion: bool = False
    nota_excepcion: Optional[str] = None
    movimiento_origen_id: Optional[UUID] = None

    def validate(self):
        if self.flag_excepcion and not self.nota_excepcion:
            raise ValidationError("nota_excepcion es obligatoria para overrides (RN-10)")
        if self.tipo == 'contra-movimiento' and not self.nota:
            raise ValidationError("nota es obligatoria para contra-movimientos (RN-10)")
        if self.fecha_hora_evento > datetime.now():
            raise ValidationError("La fecha/hora del evento no puede ser futura (RN-08)")
```

---

## 5. Singleton para configuración global (Creacional) — uso limitado

La lista de categorías y las constantes de negocio (umbral de retroactividad = 1 hora, período de historial de alertas = 30 días) se cargan una vez al iniciar la aplicación. No requieren un Singleton explícito — en FastAPI esto se maneja con la carga de la configuración al arrancar.

---

## Patrones que NO se aplican aquí

| Patrón | Por qué no |
|--------|-----------|
| **CQRS** | Separa modelos de lectura y escritura. Útil cuando las queries de lectura son radicalmente diferentes de las escrituras. Aquí el modelo es el mismo: un producto con sus movimientos. |
| **Event Sourcing** | Reconstruir el estado desde eventos tiene valor para auditoría completa. Aquí los movimientos ya son el log de eventos — el estado actual (stock) se calcula desde ellos, que es exactamente lo que hace Event Sourcing, pero sin su complejidad adicional. |
| **Saga** | Para transacciones distribuidas entre microservicios. Con un monolito y una sola BD, las transacciones ACID reemplazan toda la complejidad de Saga. |
| **Factory** | No hay variedad de tipos de objetos que requieran creación parametrizable compleja. El Command Pattern cubre la creación de movimientos. |
