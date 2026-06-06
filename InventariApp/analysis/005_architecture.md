# Arquitectura del Sistema — InventariApp

## Arquitectura recomendada: Monolito por capas con núcleo de ledger inmutable

Un monolito bien estructurado en capas: presentación (API REST), dominio (lógica de negocio), y datos (repositorios + PostgreSQL). El núcleo del dominio es un **ledger de movimientos inmutable**: toda la verdad del estado del inventario deriva del historial append-only de entradas y salidas.

---

## Justificación

Dos criterios dominan la decisión:

**1. La escala es pequeña y acotada.** 18 empleados, 1 almacén, un único operador de movimientos (Andrés). El pico de concurrencia realista es 3-4 usuarios simultáneos. Un monolito en un servidor con 1 CPU y 512MB de RAM maneja esto sin esfuerzo.

**2. La lógica del dominio es transaccional y acoplada.** Cuando Andrés registra una salida, deben ocurrir en la misma transacción:
1. Insertar el movimiento en el historial
2. Recalcular el stock actual del producto
3. Evaluar si se debe generar o resolver una alerta de stock bajo
4. (Si la salida es un override) Registrar la excepción

Si estas cuatro operaciones no son atómicas, puede quedar una alerta sin generar, una alerta sin resolver, o un stock inconsistente con el historial. En un monolito con una sola base de datos relacional, esto es un `BEGIN/COMMIT`. En microservicios requeriría el patrón Saga — complejidad de orden de magnitud superior sin ningún beneficio a esta escala.

**El insight central del dominio:** el stock de un producto nunca se almacena directamente — se calcula siempre desde la suma de sus movimientos. Esto hace que el sistema sea un **ledger** (libro contable), no una base de datos mutable de stocks.

*Como se aprendió en Contador_Calorias con entidades calculadas: los valores derivables de la fuente de verdad no deben almacenarse por separado. Aquí aplica con fuerza institucional: el stock actual es siempre `SUM(movimientos)`, nunca un campo que alguien pueda editar directamente.*

---

## Alternativas consideradas

### Opción A: Microservicios (Inventario / Alertas / Órdenes como servicios separados)
- **¿Cuándo sería correcto?**: si hubiera equipos independientes, múltiples almacenes, y necesidad de escalar componentes de forma diferente.
- **¿Por qué se descarta?**: la lógica de generación de alertas está acoplada atómicamente con la inserción de movimientos. Separarlos requiere mensajería asíncrona con garantías de exactly-once — complejidad enorme para 18 usuarios.

### Opción B: BaaS (Firebase/Supabase) sin backend propio
- **¿Cuándo sería correcto?**: MVPs rápidos sin lógica transaccional compleja.
- **¿Por qué se descarta?**: la evaluación de alertas al registrar un movimiento (generar/resolver automáticamente) requiere lógica server-side que no puede vivir en reglas de seguridad de Firebase. Terminaría siendo Cloud Functions — un mini-backend de todas formas.

### Opción C: Serverless (Lambda por función)
- **¿Cuándo sería correcto?**: tráfico muy variable con picos impredecibles.
- **¿Por qué se descarta?**: las transacciones de BD con locking requieren conexiones persistentes. El cold start introduce latencia. Para tráfico constante de una empresa pequeña, un proceso siempre activo es más simple y eficiente.

---

## Componentes principales

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CLIENTE (Browser)                                │
│  React SPA — web responsive                                         │
│  Vistas: Dashboard/KPIs | Registro de movimientos |                 │
│          Catálogo de productos | Historial | Órdenes | Excepciones  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTPS / REST API (JSON)
┌──────────────────────────▼──────────────────────────────────────────┐
│                      BACKEND MONOLITO                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Capa API (Controladores REST)                               │   │
│  │  - Endpoints de movimientos, productos, alertas, órdenes     │   │
│  │  - Serialización/validación de requests (Pydantic)           │   │
│  └──────────────────────┬──────────────────────────────────────┘   │
│                         │                                           │
│  ┌──────────────────────▼──────────────────────────────────────┐   │
│  │  Capa de Dominio                                             │   │
│  │  - MovementService: registrar movimiento (entrada/salida)    │   │
│  │  - AlertEvaluator: generar/resolver alertas automáticamente  │   │
│  │  - StockCalculator: SUM(movimientos) por producto            │   │
│  │  - OrderService: gestionar ciclo de vida de órdenes          │   │
│  │  - OverridePolicy: bloquear o autorizar salidas sin stock    │   │
│  └──────────────────────┬──────────────────────────────────────┘   │
│                         │                                           │
│  ┌──────────────────────▼──────────────────────────────────────┐   │
│  │  Capa de Datos (Repositories)                                │   │
│  │  - MovementRepository (append-only)                          │   │
│  │  - ProductRepository                                         │   │
│  │  - AlertRepository                                           │   │
│  │  - OrderRepository                                           │   │
│  └──────────────────────┬──────────────────────────────────────┘   │
└──────────────────────────┼──────────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────────┐
│                       PostgreSQL                                    │
│  productos | movimientos | alertas_stock_bajo |                     │
│  ordenes_reposicion | proveedores | categorias                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Trade-offs

| Ganamos | Sacrificamos |
|---------|-------------|
| Transacciones ACID para movimiento + alerta en un solo commit | Escalabilidad independiente de componentes |
| Historial inmutable = auditoría garantizada por diseño | Complejidad del stock calculado vs. almacenado (mitigable con índices) |
| Un solo proceso que operar y monitorear | Si el proceso cae, todo cae (mitigable con Docker restart policy) |
| La lógica de negocio vive en un solo lugar | Un monolito grande puede volverse difícil de mantener si el dominio crece mucho |

---

## Ejemplos reales

- **QuickBooks / sistemas ERP pequeños**: el inventario en sistemas contables siempre opera como un ledger — los ajustes de stock son asientos contables, nunca ediciones directas al saldo.
- **Basecamp**: monolito deliberado para una empresa de tamaño similar. DHH defiende públicamente que la complejidad operativa de los microservicios no vale la pena para equipos y escalas como esta.
- **Shopify (inicios)**: monolito Rails para miles de tiendas. El monolito fue la base sólida sobre la que construyeron antes de extraer servicios específicos bajo demanda real.
