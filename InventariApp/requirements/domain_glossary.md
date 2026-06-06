# Domain Glossary — 010 Discovery
Proyecto: Sistema de Inventario y Alertas de Stock
Empresa: Distribuidora Andina Ltda.
Generado: 2026-06-01T19:12:09Z

Terminos extraidos del transcript de entrevistas y del brief. Se usa la terminologia tal como la usan los stakeholders.

---

## Terminos del Dominio

| Termino | Definicion segun stakeholders | Fuente |
|---------|------------------------------|--------|
| Almacen / Bodega | El espacio fisico donde se guardan los productos. Andres usa ambos terminos indistintamente. | S-01 (R1, R2) |
| Entrada | Movimiento que aumenta el stock: compra a proveedor o devolucion de cliente. | Brief F-02, S-01 |
| Salida | Movimiento que disminuye el stock: despacho a cliente o perdida/dano. | Brief F-02, S-01 |
| Contra-movimiento | Movimiento de correccion que revierte una entrada o salida erronea. No se borra el movimiento original; se registra uno nuevo opuesto. Andres lo describe como la forma de corregir sin perder el rastro. | S-01 (R3) |
| Stock actual | Cantidad de un producto disponible en el almacen en este momento. Se calcula siempre a partir del historial de movimientos; nunca se edita directamente. | Brief F-01, S-01 |
| Stock minimo | Umbral configurado por Diana debajo del cual el sistema genera una alerta. Cada producto tiene el suyo. Puede cambiar por temporada. | Brief F-01, S-02 (R7, R8) |
| Stock objetivo | Stock al que se quiere llegar al reponer. El brief lo define como: stock minimo multiplicado por 2. | Brief F-04 |
| Alerta de stock bajo | Notificacion visible en la pantalla principal cuando el stock de un producto llega o cae por debajo del stock minimo. Persiste hasta que el stock supera el minimo. No llega por correo ni push. | Brief F-03, S-01 (R6), S-02 (R2) |
| Alerta activa | Una alerta de stock bajo que aun no se ha resuelto porque el stock no ha superado el minimo. | Brief F-03 |
| Excepcion / Despacho en rojo | Salida registrada cuando el stock disponible es insuficiente, autorizada explicitamente. Requiere nota obligatoria con el motivo. Ocurre ~2-3 veces al mes cuando Diana sabe que llega mercancia ese dia. | S-01 (R4, R5), S-02 (R3) |
| Override | Accion de forzar una salida sobre stock insuficiente. Ver Excepcion. | S-01 (R4) |
| Historial de excepciones | Registro de todas las salidas forzadas sobre stock, con su nota de motivo. Consultable por Diana para detectar patrones de reorden mal calibrado. | S-02 (R3) |
| Orden de reposicion | Solicitud formal de compra generada por Diana desde una alerta activa. Tiene estados: Borrador, Confirmada, Recibida. Al recibirla, el sistema registra automaticamente la entrada. | Brief F-04, S-02 (R4) |
| Punto de reorden | Termino usado por Diana para referirse al stock minimo. Equivalente. | S-02 (R3) |
| KPI ejecutivo | Indicador clave de desempeno que Luis necesita ver al entrar al sistema. En el contexto de este proyecto: (1) productos en alerta activa y (2) despachos con excepcion en el mes. Son conteos, no valores monetarios. | S-03 (R2) |
| Datos no conciliados | Condicion en la que el sistema detecta que el inventario calculado podria no reflejar la realidad (ej: movimientos retroactivos pendientes de validar, descuadres). El panel ejecutivo debe advertir esto antes de mostrar los KPIs. | S-03 (R4) |
| Descuadre | Diferencia entre el stock que el sistema registra y el stock fisico real. Antes del sistema nuevo, los descuadres requerianconteos fisicos de medio dia para resolverse. | S-01 (R3), S-03 (R1) |
| Camion / Mercancia en camino | Expresion usada por los stakeholders para justificar un despacho en rojo: saben que el proveedor esta entregando ese dia aunque el sistema no lo refleje aun. | S-01 (R4, R5) |
| Proveedor principal | El proveedor habitual de un producto. Cada producto puede tener hasta 2 proveedores alternativos registrados. | Brief F-01, F-04 |
| Discontinuado | Estado de un producto que ya no recibe nuevas entradas. Su historial de movimientos se preserva. No aparece en el catalogo activo. | Brief RN-03 |
| Codigo interno | Identificador alfanumerico unico e inmutable de un producto. Se asigna al crearlo y no puede cambiarse. | Brief RN-04 |
| Unidad de medida | Como se cuenta un producto: unidad, caja, resma, cartucho. | Brief F-01 |
| Panel de estado general | Vista de resumen para Luis con los KPIs y la tabla de todos los productos. | Brief F-06 |
| Temporada alta | Periodo del ano (ej: diciembre) en que ciertos productos rotan significativamente mas. Diana ajusta los stocks minimos por temporada. | S-02 (R8) |
| Autoservicio | Capacidad de Diana de configurar los stocks minimos directamente en la interfaz, sin depender de un administrador de sistemas o un ticket. | S-02 (R8) |
| Web responsive | Diseno de la interfaz que se adapta a pantallas de distintos tamanos, permitiendo acceso desde dispositivo movil sin necesidad de app nativa. Resuelve la necesidad de Andres de registrar movimientos desde la bodega. | S-01 (R1-R2), Brief (plataforma) |
| Inmovilizacion de capital | Termino usado por Diana para referirse al costo de tener exceso de stock de un producto que no se necesitaba. Ejemplo: pedido duplicado por informacion desactualizada. | S-02 (R1) |
