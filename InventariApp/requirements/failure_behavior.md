# Failure Behavior — 010 Discovery
Proyecto: Sistema de Inventario y Alertas de Stock
Empresa: Distribuidora Andina Ltda.
Generado: 2026-06-01T19:12:09Z
Fuentes: discovery/dialogue_transcript.md + inputs/brief.md

---

## 1. Comportamientos ante Fallos — Declarados por Stakeholders

Los siguientes escenarios fueron capturados directamente en las entrevistas de discovery.

| ID | Escenario | Actor/modulo afectado | Fuente | Comportamiento esperado |
|----|-----------|----------------------|--------|------------------------|
| F-01 | Error de registro: movimiento con datos incorrectos ingresado por error | Modulo de movimientos | S-01 (R3) | No borrar ni editar el movimiento original. El usuario registra un contra-movimiento (entrada que revierte una salida, o salida que revierte una entrada). Debe quedar registro visible de que hubo una correccion y cual fue. |
| F-02 | Intento de salida que supera el stock disponible actual | Modulo de movimientos | S-01 (R4), S-02 (R3) | Bloquear por defecto. Mostrar mensaje de error con el stock disponible actual. Permitir forzar la salida (override) con nota obligatoria de motivo. La excepcion queda registrada en historial de excepciones consultable por Diana. |
| F-03 | Caida del sistema inferior a 1 hora | Sistema completo | S-02 (R6) | Aceptable. El negocio opera temporalmente en papel o hoja de calculo. Al recuperarse el sistema, los movimientos se ingresan manualmente y se concilian. No es un escenario critico siempre que no se pierdan datos. |
| F-04 | Caida del sistema igual o mayor a 1 dia | Sistema completo | S-02 (R6) | Critico. El sistema debe permitir ingresar movimientos con fecha/hora retroactiva al recuperarse para no generar huecos en el historial. La integridad del historial es no negociable. Lo peor no es la caida sino que genere un hueco irrecuperable. |
| F-05 | Dato presentado en reunion ejecutiva con posible inconsistencia | Panel gerencial | S-03 (R4) | El sistema muestra advertencia visible del tipo datos no conciliados antes de presentar los KPIs. Luis prefiere no ver un numero incorrecto con apariencia de certeza. El criterio tecnico exacto de cuando se dispara esta advertencia es TBD en especificacion (ver DD-02 en scope_boundaries.md). |

---

## 2. Comportamientos ante Casos de Borde — Definidos en el Brief

Los siguientes comportamientos provienen del brief y no generaron contradiccion en el transcript. Se incluyen aqui como referencia de implementacion.

| ID | Escenario | Comportamiento definido |
|----|-----------|------------------------|
| B-01 | Salida que deja el stock exactamente en 0 | Se permite. Si el stock minimo es mayor que 0, se genera la alerta en ese momento. |
| B-02 | Producto sin movimientos registrados | El stock actual es 0. Sin historial que mostrar. En el panel de estado aparece con estado SIN MOVIMIENTOS (distinto de ALERTA, para no generar ruido si el stock minimo tambien es 0). |
| B-03 | Proveedor sin datos de contacto completos | Se permite crear un proveedor solo con nombre. Campos de contacto (telefono, email) son opcionales. Una orden de reposicion puede generarse igualmente; Diana gestiona el contacto fuera del sistema. |
| B-04 | Intento de registrar entrada a producto discontinuado | El sistema bloquea la accion y muestra mensaje explicativo. No se crea el movimiento. |
| B-05 | Orden de reposicion con cantidad ajustada a 0 por Diana | El sistema no permite confirmar una orden con cantidad 0. Muestra error de validacion. |
| B-06 | Dos alertas simultaneas del mismo producto | No ocurre por diseno. Si el stock baja mas mientras la alerta ya esta activa, la alerta existente persiste y refleja el stock actualizado. No se genera una segunda alerta para el mismo producto. |
| B-07 | Discontinuar un producto con una orden de reposicion Confirmada pendiente | El sistema lo permite pero muestra advertencia antes de confirmar la discontinuacion. La orden queda en estado Confirmada; Diana debe cancelarla manualmente. |
| B-08 | Stock minimo modificado mientras hay una alerta activa | Si el nuevo minimo es menor o igual al stock actual: la alerta se resuelve automaticamente. Si el nuevo minimo es mayor al stock actual: la alerta persiste. |

---

## 3. Nota sobre RN-02 — Stock Minimo = 0

El brief define que un producto con stock minimo configurado en 0 nunca genera alerta de stock bajo (RN-02). Este comportamiento no fue validado explicitamente con los stakeholders durante las entrevistas.

Riesgo identificado (G-03 en analysis_report.md): si un producto tiene stock minimo = 0 por omision o error, nunca generara alerta aunque el stock llegue a 0, lo que podria generar desabastecimiento silencioso. Se recomienda que la especificacion aclare si debe haber una advertencia UI cuando el usuario configura stock minimo = 0.
