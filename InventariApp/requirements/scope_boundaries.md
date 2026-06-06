# Scope Boundaries — 010 Discovery
Proyecto: Sistema de Inventario y Alertas de Stock
Empresa: Distribuidora Andina Ltda.
Generado: 2026-06-01T19:12:09Z

---

## 1. In-Scope — Capacidades Confirmadas

Las siguientes capacidades estan dentro del alcance de la v1:

| ID | Capacidad | Fuente de confirmacion |
|----|-----------|----------------------|
| IS-01 | Catalogo de productos con codigo unico inmutable, categoria, unidad de medida, stock minimo configurable y hasta 2 proveedores | Brief F-01 + confirmado por stakeholders |
| IS-02 | Registro inmutable de movimientos (entradas y salidas) con campo fecha/hora editable por el usuario | Brief F-02 + RF-E02 (S-02 R6) |
| IS-03 | Contra-movimientos para correccion de errores (sin borrado ni edicion de movimientos) | Brief F-02 + S-01 (R3) |
| IS-04 | Alertas de stock bajo visibles en pantalla principal cuando stock llega o cae bajo el minimo | Brief F-03 + RF-E03 (S-01 R6) |
| IS-05 | Vista de alerta con contexto: stock actual, stock minimo, movimientos ultimos 30 dias | RF-E04 (S-02 R4-R5) |
| IS-06 | Override de salida sobre stock con nota obligatoria + historial de excepciones consultable | RF-E01 (S-01 R4-R5, S-02 R3) — resolucion de C-02 |
| IS-07 | Autoservicio de configuracion de stock minimo por producto para Diana | RF-E05 (S-02 R8) |
| IS-08 | Ordenes de reposicion con estados Borrador/Confirmada/Recibida y creacion automatica de movimiento de Entrada al recibir | Brief F-04 |
| IS-09 | Historial completo de movimientos por producto en orden cronologico descendente | Brief F-05 + S-01 (R7) |
| IS-10 | Panel de estado general con KPIs ejecutivos (alertas activas + excepciones del mes) y tabla de productos | Brief F-06 + RF-E06 (S-03 R2) |
| IS-11 | Advertencia visible de datos no conciliados en panel ejecutivo antes de KPIs | RF-E07 (S-03 R4) — criterio de disparo TBD en especificacion |
| IS-12 | Diseno web responsive para acceso desde dispositivo movil | RF-E08 (S-01 R1-R2) — resolucion de C-01 |
| IS-13 | Productos discontinuables con historial preservado | Brief RN-03 |

---

## 2. Out-of-Scope — Exclusiones Confirmadas

| Exclusion | Razon / fuente de confirmacion |
|-----------|-------------------------------|
| Precios, costos, valoracion monetaria del inventario | Luis inicialmente lo pidio pero lo retiro al entender que los precios no viven en el sistema de forma confiable (C-03) |
| App movil nativa | Brief lo excluye; Andres acepto que web responsive resuelve su necesidad (C-01) |
| Autenticacion diferenciada por usuario (roles/login) | Brief lo excluye; Luis lo acepto para v1 aunque lo senalo como necesidad futura (S-03 R5) |
| Notificaciones push ni email | Brief lo excluye; Andres prefiere alertas en pantalla (S-01 R6) |
| Exportacion de reportes / generacion de PDF | Brief lo excluye; sin contradiccion en transcript |
| Multi-almacen o ubicaciones dentro del almacen | Brief lo excluye; sin contradiccion en transcript |
| Gestion de lotes ni fechas de vencimiento | Brief lo excluye; sin contradiccion en transcript |
| Lectores de codigo de barras o QR | Brief lo excluye; sin contradiccion en transcript |
| Flujo de aprobacion de ordenes de reposicion | Brief lo excluye; Diana confirma directamente |
| Gestion de proveedores, contactos o pedidos | Brief lo excluye; Diana lo maneja fuera del sistema (S-02 R4) |

---

## 3. Decisiones de Diseno Pendientes (Ambiguedades a Resolver en Especificacion)

| ID | Area | Descripcion | Impacto |
|----|------|-------------|---------|
| DD-01 (A-01) | Campo fecha/hora en movimientos | IS-02 establece que la fecha es editable para permitir movimientos retroactivos. La especificacion debe definir: rangos de fecha permitidos, si hay validacion de fecha futura, y si hay un log de auditoria que muestre que el usuario cambio la fecha. | Medio — afecta diseno del formulario de movimiento y auditoria |
| DD-02 (A-02) | Criterio de disparo de advertencia datos no conciliados | IS-11 establece que el panel ejecutivo muestra esta advertencia, pero no hay criterio operacional definido de cuando se dispara. La especificacion debe definir que condicion tecnica activa la advertencia. | Bajo-Medio — funcionalidad deseable; no bloquea el diseno principal |
| DD-03 (A-03) | Interfaz de acceso al historial de excepciones para Diana | IS-06 establece que Diana puede consultar el historial de excepciones. La especificacion debe decidir si este historial es accesible desde la vista de alertas, desde el historial del producto, o desde una vista dedicada. | Bajo — decision de navegacion UI |

---

## 4. Riesgos Conocidos

| ID | Riesgo | Descripcion | Impacto estimado |
|----|--------|-------------|-----------------|
| R-01 (G-01) | Control de acceso sin diferenciacion | Sin autenticacion diferenciada, cualquier usuario puede modificar cualquier dato. Luis acepto el riesgo para v1 pero senalo que si alguien cambia algo sin querer, puede generar descuadres dificiles de rastrear. | Bajo para v1; a monitorear para v2 |
| R-02 (G-02) | Ausencia de notificacion a Diana al realizar override | Cuando Andres fuerza una salida con nota, Diana solo se entera si revisa el historial de excepciones activamente. No hay notificacion automatica. Podria generar situaciones donde Diana no sabe que hubo un override hasta despues. | Bajo; aceptado dado que las excepciones son 2-3 veces/mes |
| R-03 (G-03) | Comportamiento stock minimo = 0 sin validacion de stakeholder | El brief define RN-02: stock minimo=0 nunca genera alerta. Este comportamiento no fue validado explicitamente con los stakeholders. Si un producto tiene minimo=0 por error y nunca genera alerta, podria agotarse sin aviso. | Bajo; el riesgo operativo es bajo si la mayoria de productos tienen minimo > 0 |
