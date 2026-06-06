# Estrategia de Testing — InventariApp

## Pirámide de testing aplicada

```
        /\
       /E2E\          ~10% — Flujos completos (Playwright)
      /──────\
     /Integr. \       ~30% — Transacciones con BD real (PostgreSQL en Docker)
    /────────── \
   / Unit Tests  \    ~60% — Lógica de dominio pura (AlertEvaluator, StockCalculator)
  /______________  \
```

La proporción es similar a MeetingApp: la capa de integración tiene peso relevante porque la lógica más crítica (evaluación de alertas, transaccionalidad de movimientos) solo puede verificarse completamente contra una BD real.

*Como se aprendió en MeetingApp: los tests de integración con BD real son necesarios para verificar comportamiento transaccional. Un mock del repositorio no puede verificar que la alerta se genera en la misma transacción que el movimiento.*

---

## Qué testear obligatoriamente

### 1. Lógica de evaluación de alertas — Prioridad MÁXIMA

Esta lógica es el corazón del sistema. Una regresión aquí significa alertas que no se generan (stock agotado sin aviso) o alertas fantasma (alertas activas cuando el stock está bien).

```python
class TestAlertEvaluator:
    # Generación de alerta
    def test_alerta_generada_cuando_stock_llega_al_minimo(self): ...
    def test_alerta_generada_cuando_stock_cae_bajo_el_minimo(self): ...
    def test_alerta_generada_cuando_stock_llega_a_cero_con_minimo_positivo(self): ...
    def test_sin_alerta_cuando_stock_cero_y_minimo_cero(self):    # RN-02, RN-16
    def test_sin_segunda_alerta_si_ya_hay_una_activa(self):       # RN-13

    # Resolución de alerta
    def test_alerta_resuelta_cuando_stock_supera_minimo(self):    # RN-15
    def test_alerta_no_resuelta_cuando_stock_igual_al_minimo(self):
    def test_alerta_resuelta_al_cambiar_minimo_a_cero(self):      # SE-07

    # Caso edge: stock mínimo = 0
    def test_advertencia_ui_al_configurar_minimo_en_cero(self):   # SE-08
    def test_no_genera_alerta_con_minimo_cero_aunque_stock_sea_cero(self):
```

### 2. Cálculo de stock — Prioridad ALTA

```python
class TestStockCalculator:
    def test_stock_inicial_es_cero_sin_movimientos(self):
    def test_entrada_incrementa_stock(self):
    def test_salida_decrementa_stock(self):
    def test_contra_movimiento_revierte_salida(self):       # SC-03
    def test_contra_movimiento_revierte_entrada(self):
    def test_override_puede_dejar_stock_negativo(self):     # AC-F15-05
    def test_stock_es_suma_de_todos_los_movimientos(self):  # Invariante fundamental

    # Caso edge del ledger
    def test_stock_con_cien_movimientos_alternados(self):
    def test_stock_consistente_despues_de_conciliacion_retroactiva(self):
```

### 3. Reglas de negocio de movimientos — Prioridad ALTA

```python
class TestMovementRules:
    def test_bloqueo_entrada_a_producto_discontinuado(self):     # RN-03, SE-05
    def test_bloqueo_fecha_futura(self):                         # RN-08, SE-06
    def test_flag_retroactivo_con_diferencia_mayor_1_hora(self): # RN-09, SE-03
    def test_sin_flag_retroactivo_diferencia_menor_1_hora(self):
    def test_nota_obligatoria_en_override(self):                 # RN-10, SE-02
    def test_nota_obligatoria_en_contra_movimiento(self):        # RN-10, SC-03
    def test_orden_recibida_genera_movimiento_entrada(self):     # RN-18, SC-12
```

### 4. Override (despacho en rojo) — Prioridad ALTA

```python
class TestOverridePolicy:
    def test_bloqueo_por_defecto_cuando_stock_insuficiente(self):      # SE-01
    def test_override_permitido_con_nota_obligatoria(self):            # SE-02
    def test_override_registra_flag_excepcion(self):
    def test_override_aparece_en_historial_excepciones(self):          # RN-11
    def test_override_puede_dejar_stock_negativo(self):               # AC-F15-05
    def test_sin_nota_override_falla_validacion(self):
```

### 5. Ciclo de vida de órdenes — Prioridad MEDIA

```python
class TestOrderLifecycle:
    def test_transicion_borrador_a_confirmada(self):            # SC-11
    def test_transicion_confirmada_a_recibida_crea_movimiento(self): # SC-12, RN-18
    def test_transicion_invalida_confirmada_a_borrador(self):
    def test_no_confirmar_con_cantidad_cero(self):              # RN-17, SE-09
    def test_advertencia_al_discontinuar_con_orden_confirmada(self): # SE-10
```

### 6. Tests de integración: transaccionalidad

Este es el caso que los unit tests NO pueden cubrir:

```python
class TestTransactionality:
    def test_movimiento_y_alerta_en_misma_transaccion(self):
        """
        Si el INSERT del movimiento tiene éxito pero la generación de alerta falla,
        ambas deben revertirse. El stock no debe cambiar sin que la alerta refleje el nuevo estado.
        """
        # Simular fallo en la creación de alerta
        # Verificar que el movimiento también fue revertido
        ...

    def test_concurrencia_no_genera_alertas_duplicadas(self):
        """
        Dos movimientos simultáneos sobre el mismo producto no deben crear
        dos alertas activas (viola RN-13).
        """
        import threading
        results = []
        def register_movement():
            results.append(service.register_movement(...))

        t1 = threading.Thread(target=register_movement)
        t2 = threading.Thread(target=register_movement)
        t1.start(); t2.start()
        t1.join(); t2.join()

        active_alerts = repo.get_all_active_alerts(producto_id)
        assert len(active_alerts) == 1, "Solo debe existir una alerta activa"
```

---

## Qué no vale la pena testear

| Qué | Por qué no |
|-----|-----------|
| Que SQLAlchemy previene SQL injection | Es comportamiento de la librería |
| Que PostgreSQL respeta las FOREIGN KEY constraints | Es comportamiento de PostgreSQL |
| Que Pydantic valida tipos de datos | Es comportamiento de Pydantic |
| Que el `timestamp_creacion` es correcto | Está garantizado por `DEFAULT NOW()` en la BD |

---

## Testing de la base de datos

**Estrategia**: PostgreSQL real en Docker durante los tests de integración.

```yaml
# docker-compose.test.yml
services:
  test-db:
    image: postgres:16
    environment:
      POSTGRES_DB: inventariapp_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
```

- `beforeEach`: truncar tablas con `TRUNCATE ... CASCADE RESTART IDENTITY`
- `afterAll`: derribar el contenedor

**¿Por qué no SQLite para tests?** SQLite no implementa `SELECT FOR UPDATE`, ni el índice parcial `WHERE estado='activa'`, ni `TIMESTAMPTZ`. Los tests de concurrencia y las constraints de alerta única requieren PostgreSQL real.

---

## Estrategia de datos de prueba

```python
# Factory Pattern para datos de prueba — sin fixtures compartidas entre tests
def make_producto(stock_minimo=10, estado='activo', **kwargs) -> Producto:
    return Producto(
        codigo_interno=f"TEST-{uuid4().hex[:6].upper()}",
        nombre="Producto de prueba",
        categoria_id=make_categoria().id,
        unidad_medida="unidad",
        stock_minimo=stock_minimo,
        estado=estado,
        **kwargs
    )

def make_movimiento(producto_id, tipo_impacto='SUMA', cantidad=10, **kwargs) -> Movimiento:
    return Movimiento(
        producto_id=producto_id,
        tipo='entrada',
        tipo_impacto=tipo_impacto,
        cantidad=cantidad,
        fecha_hora_evento=datetime.now() - timedelta(minutes=5),  # siempre pasado
        **kwargs
    )
```

Cada test crea sus propios datos — sin estado compartido entre tests de alertas ni de stock.
