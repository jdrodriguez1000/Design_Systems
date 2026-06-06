# Estrategia de Testing — MeetingApp

## Pirámide de testing aplicada

```
        /\
       /E2E\          ~10% — Flujos completos (Playwright)
      /──────\
     /Integr. \       ~30% — Lógica + BD real (PostgreSQL en Docker)
    /────────── \
   / Unit Tests  \    ~60% — Lógica de dominio pura (sin BD, sin red)
  /______________  \
```

A diferencia de Contador_Calorias (donde el 70% eran unit tests de lógica pura), aquí la capa de integración tiene más peso porque la lógica crítica — detección de solapamiento, locking, transacciones — solo puede verificarse correctamente contra una BD real. Un mock del repositorio no puede detectar una race condition.

---

## Qué testear obligatoriamente

### 1. Detección de solapamiento de intervalos — Prioridad MÁXIMA

Esta lógica es el corazón del sistema. Una regresión aquí significa doble-booking.

```python
class TestIntervalOverlap:
    # Los 7 casos de solapamiento posibles
    def test_no_overlap_before(self):        # nueva termina antes de que empiece la existente
    def test_no_overlap_after(self):         # nueva empieza después de que termina la existente
    def test_partial_overlap_start(self):    # nueva empieza antes y termina en medio
    def test_partial_overlap_end(self):      # nueva empieza en medio y termina después
    def test_complete_overlap_incoming(self):# nueva está completamente dentro de la existente
    def test_complete_overlap_existing(self):# existente está completamente dentro de la nueva
    def test_exact_same_interval(self):      # exactamente el mismo horario → conflicto
    def test_adjacent_no_overlap(self):      # nueva empieza exactamente cuando termina la existente → NO conflicto
```

### 2. Lógica de prioridad y desplazamiento — Prioridad ALTA

```python
class TestDisplacement:
    def test_standard_displaced_by_prioritaria(self):          # ACP-17
    def test_prioritaria_not_displaced_by_standard(self):      # RN-03
    def test_same_priority_oldest_wins(self):                   # ACP-19, RN-04
    def test_displacement_generates_notification(self):         # RN-23
    def test_displacement_notification_includes_alternative(self): # ACP-18
    def test_displacement_notification_without_alternative(self):  # ACP-18 (sin sala libre)
    def test_automatic_priority_for_client_externo(self):      # ACP-15, RN-07
    def test_automatic_priority_for_directorio(self):          # ACP-15, RN-07
    def test_only_gerente_can_mark_urgent(self):                # ACP-16, RN-13
```

### 3. Regla de exclusividad por usuario — Prioridad ALTA

```python
class TestUserExclusivity:
    def test_user_cannot_overlap_own_reservations(self):   # ACP-07, RN-01
    def test_rule_applies_to_all_roles(self):               # ACP-14, ACP-21, RN-15
    def test_retroactiva_also_blocked_if_own_overlap(self): # ACP-14, SE-06
```

### 4. Reservas retroactivas — Prioridad ALTA

```python
class TestRetroactiveReservations:
    def test_only_admin_can_create_retroactive(self):         # ACP-10, RN-05
    def test_retroactive_without_conflict_creates_ok(self):   # ACP-10
    def test_retroactive_with_conflict_shows_warning(self):   # ACP-11, EP-05
    def test_retroactive_with_conflict_can_be_confirmed(self):# ACP-13 — alerta NO bloquea
```

### 5. Tests de integración con BD (race conditions)

Este es el caso que los unit tests NO pueden cubrir:

```python
class TestConcurrentBooking:
    def test_two_concurrent_requests_only_one_succeeds(self):
        """
        Lanza 2 threads que intentan reservar la misma sala al mismo tiempo.
        Solo uno debe tener éxito. Sin SELECT FOR UPDATE, ambos podrían ver
        la sala como disponible antes de que el otro haga el INSERT.
        """
        import threading
        results = []
        def book():
            try:
                results.append(('success', service.create_reservation(...)))
            except ConflictError as e:
                results.append(('conflict', e))

        t1, t2 = threading.Thread(target=book), threading.Thread(target=book)
        t1.start(); t2.start()
        t1.join(); t2.join()

        successes = [r for r in results if r[0] == 'success']
        assert len(successes) == 1, "Exactamente una reserva debe tener éxito"
```

---

## Testing de la base de datos

**Estrategia**: base de datos PostgreSQL real en Docker durante los tests de integración.

```yaml
# docker-compose.test.yml
services:
  test-db:
    image: postgres:16
    environment:
      POSTGRES_DB: meetingapp_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
```

- `beforeEach`: truncar tablas (más rápido que recrear el schema)
- `afterAll`: derribar el contenedor

**¿Por qué no un mock del repositorio para estos tests?**

Para verificar que `SELECT FOR UPDATE` funciona correctamente, se necesita una BD real que implemente el locking. Un mock en memoria no tiene este comportamiento.

---

## Qué no vale la pena testear

| Qué | Por qué no |
|-----|-----------|
| El ORM/queries parametrizadas previenen SQL injection | Es comportamiento de la librería, no del código de la app |
| Que bcrypt hashea correctamente | Es comportamiento de bcrypt, no del código |
| Que PostgreSQL respeta las FOREIGN KEY constraints | Es comportamiento de PostgreSQL |
| Serialización/deserialización de JSON | Es comportamiento del framework |

---

## Testing de rendimiento

Para V1 con 25 usuarios: no necesario. Si la empresa crece:
- Simular 50 requests concurrentes de disponibilidad con Locust o k6
- El cuello de botella esperado: la query de solapamiento sin índice

---

## Estrategia de datos de prueba

```python
# Factory pattern para datos de prueba — no fixtures compartidas
def make_reserva(sala_id=None, inicio=None, fin=None, prioridad='estandar', **kwargs):
    return Reserva(
        sala_id=sala_id or make_sala().id,
        fecha_inicio=inicio or datetime.now() + timedelta(hours=1),
        fecha_fin=fin or datetime.now() + timedelta(hours=2),
        tipo_prioridad=prioridad,
        **kwargs
    )
```

Cada test crea sus propios datos — sin estado compartido entre tests. Esto es especialmente importante para los tests de concurrencia y desplazamiento, donde el estado de la BD debe ser predecible.

*Como se aprendió en Contador_Calorias: las funciones puras son trivialmente testeables. Aquí aplica lo mismo: `ConflictResolver.resolve(incoming, existing)` es una función pura que no necesita BD para testearse.*
