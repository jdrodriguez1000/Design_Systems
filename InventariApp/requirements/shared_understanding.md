# Shared Understanding Document
Proyecto: Sistema de Inventario y Alertas de Stock
Empresa: Distribuidora Andina Ltda.
Generado: 2026-06-01T19:12:09Z
Fase: 010 Discovery

---

## 1. Contexto del Negocio

Distribuidora Andina Ltda. es una empresa de 18 personas dedicada a la distribucion de insumos de oficina. Opera un unico almacen fisico en Bogota. El control de inventario actual en hoja de calculo genera: quiebres de stock sin aviso previo, registros desactualizados por concurrencia, sin historial confiable de movimientos, y conteos fisicos para cuadrar descuadres.

El sistema a construir reemplaza la hoja de calculo para el control de inventario y alertas de stock.

---

## 2. Actores y Sus Objetivos de Valor

### 2.1 Andres Mora — Almacenista (usuario operativo principal)

Unico responsable del almacen. Registra todas las entradas y salidas. Trabaja desde PC en el almacen; pasa el 80% del dia en la bodega.

Objetivos de valor:
1. Registrar entradas y salidas de forma rapida desde interfaz web responsive, incluyendo acceso desde movil en la bodega.
2. Ver alertas de stock bajo en la pantalla principal al ingresar, sin tener que buscarlas.
3. Consultar historial completo de movimientos por producto para rastrear descuadres.
4. Corregir errores mediante contra-movimientos que preservan trazabilidad completa (sin borrar datos).

### 2.2 Diana Vargas — Jefa de Compras (decisora de reposicion)

Gestiona relacion con proveedores y decide que y cuanto reponer. Trabaja desde laptop, a veces fuera de la oficina. Autoriza excepcionalmente despachos cuando el stock es cero pero sabe que llega mercancia.

Objetivos de valor:
1. Ver inventario en tiempo real sin depender de actualizacion manual de Andres.
2. Recibir alertas con contexto de consumo de los ultimos 30 dias para tomar decisiones de reposicion informadas.
3. Configurar y actualizar stock minimo por producto en autoservicio (sin IT). Los minimos varian por temporada.
4. Consultar historial de excepciones autorizadas (despachos forzados sobre stock) para detectar patrones de reorden mal calibrado.
5. Acceder al sistema desde su laptop incluso fuera de la oficina.

### 2.3 Luis Pedraza — Gerente General (vision estrategica)

Ve el inventario como activo principal del negocio. Necesita visibilidad del estado general para reportar a socios.

Objetivos de valor:
1. KPI en pantalla principal: productos en alerta de stock bajo ahora mismo.
2. KPI en pantalla principal: despachos realizados con excepcion (forzando stock insuficiente) en el mes en curso.
3. Confiabilidad del dato: si hay inconsistencias internas, el sistema muestra advertencia visible antes de los KPIs, no un numero incorrecto con apariencia de certeza.

### 2.4 Vendedores (actores externos, sin interaccion directa en v1)

Solicitan productos al almacen. Objetivo derivado: que haya stock cuando prometen entregas. No tienen interfaz propia en esta version.

---

## 3. Que Hace el Sistema

### 3.1 Catalogo de productos
Nombre, codigo interno unico e inmutable, categoria, unidad de medida, stock minimo configurable por Diana, stock actual calculado a partir de movimientos (nunca editable directamente), hasta 2 proveedores por producto. Productos discontinuables con historial preservado.

### 3.2 Registro de movimientos con trazabilidad completa
Entradas (compras, devoluciones) y salidas (despachos, perdidas/dano). Cada movimiento: tipo, producto, cantidad, fecha/hora editable, referencia opcional, nota libre opcional. Historial inmutable: para corregir errores se registra un contra-movimiento. La fecha/hora editable permite movimientos retroactivos para conciliacion post-caida.

### 3.3 Control de salidas sobre stock con excepcion autorizada
Por defecto: bloqueo si la cantidad supera el stock disponible, mostrando el stock actual. Excepcion: se puede forzar la salida con nota obligatoria. La excepcion queda en historial consultable por Diana.

### 3.4 Alertas de stock bajo
Alerta activa cuando el stock llega o cae por debajo del minimo configurado. Visible en pantalla principal. Se resuelve automaticamente al superar el minimo. Vista de alerta incluye: stock actual, stock minimo, movimientos ultimos 30 dias.

### 3.5 Ordenes de reposicion
Diana genera ordenes desde una alerta activa: producto, proveedor, cantidad sugerida (stock objetivo - stock actual) ajustable. Estados: Borrador, Confirmada, Recibida. Al marcar Recibida, el sistema crea automaticamente el movimiento de Entrada.

### 3.6 Historial y trazabilidad por producto
Historial completo de movimientos de un producto: fecha, tipo, cantidad, referencia, nota. Orden cronologico descendente. Sin paginacion en esta version.

### 3.7 Panel de estado general con KPIs ejecutivos
KPIs prominentes: productos en alerta activa ahora + despachos con excepcion en el mes. Tabla de productos: stock actual, stock minimo, estado (OK / ALERTA / SIN MOVIMIENTOS), categoria. Filtrable por categoria y estado. Advertencia visible si hay datos no conciliados.

---

## 4. Que NO Hace el Sistema

Exclusiones confirmadas con los stakeholders:
- Sin precios, costos ni valoracion monetaria (Luis acepto; los precios no viven en el sistema)
- Sin app movil nativa (necesidad de Andres resuelta con web responsive)
- Sin autenticacion diferenciada por usuario (aceptada para v1 por Luis; necesidad futura)
- Sin notificaciones push ni email (Andres prefiere alertas en pantalla)
- Sin exportacion de reportes ni PDF
- Sin multi-almacen ni ubicaciones internas
- Sin gestion de lotes ni fechas de vencimiento
- Sin codigo de barras ni QR
- Sin flujo de aprobacion de ordenes (Diana confirma directamente)
- Sin gestion de proveedores ni pedidos

---

## 5. Aprobacion del Cliente

Estado: APROBADO POR CLIENTE

Documento generado a partir de entrevistas de discovery realizadas el 2026-06-01 con:
- S-01 Andres Mora (Almacenista)
- S-02 Diana Vargas (Jefa de Compras)
- S-03 Luis Pedraza (Gerente General)

Pendiente de aprobacion formal del cliente para cerrar la fase 010 Discovery.
