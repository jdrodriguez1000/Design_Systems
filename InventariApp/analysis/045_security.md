# Seguridad — InventariApp

## El contexto de seguridad es inusual: sin autenticación en v1

InventariApp v1 opera sin autenticación diferenciada. Luis lo aceptó explícitamente (R-01 en scope_boundaries.md). Esto cambia el modelo de amenazas: el sistema asume que todos los usuarios que pueden acceder a la URL son confiables. El foco de seguridad no está en "quién puede hacer qué" sino en "que el sistema no sea explotable desde el exterior".

---

## Modelo de amenazas

| Actor | Vector de ataque | Probabilidad | Impacto |
|-------|-----------------|-------------|---------|
| Usuario interno que modifica datos por error | Acceso no intencional a funciones de otro rol | Media (sin auth) | Medio — genera descuadres rastreables via historial inmutable |
| Externo que accede a la URL interna | Acceso desde internet si el servidor no está en red privada | Baja (si está en red corporativa) | Alto — puede registrar movimientos o ver inventario completo |
| SQL Injection en formularios | Inyectar SQL en nombre de producto, nota, referencia | Media si no se usan queries parametrizadas | Alto — acceso total a la BD |
| Manipulación de requests (fecha futura) | Enviar fecha/hora futura en API directamente | Media (sin auth, DevTools) | Bajo — el servidor valida la fecha |
| Borrado de movimientos via API | Llamar DELETE /movimientos/:id directamente | Media (sin auth) | Alto — viola la inmutabilidad del ledger |

---

## Autenticación: ninguna en v1 — plan claro para v2

**Decisión aceptada**: Luis confirmó que la empresa de 18 personas confía en sus empleados y el sistema operará en red interna. La ausencia de autenticación es un riesgo aceptado y documentado.

**Mitigación parcial en v1**: desplegar el sistema solo accesible desde la red interna de la empresa (VPN o IP whitelist en el servidor). Nunca exponer el sistema directamente a internet.

**Plan para v2**: cuando se implemente autenticación, la arquitectura recomendada es JWT con RBAC:

```python
# Roles futuros (v2)
PERMISSIONS = {
    'almacenista': {'registrar_movimiento', 'ver_historial', 'ver_alertas'},
    'jefa_compras': {'ver_inventario', 'configurar_stock_minimo',
                     'ver_excepciones', 'gestionar_ordenes'},
    'gerente':      {'ver_dashboard', 'ver_kpis'}
}
```

*Como se aprendió en MeetingApp: la verificación de permisos SIEMPRE debe ocurrir en el servidor, en cada request. La UI que oculta botones es UX, no seguridad. Este principio aplica desde el primer día del diseño, aunque en v1 todos los usuarios tengan acceso completo.*

---

## La seguridad inherente del ledger inmutable

El diseño append-only de la tabla `movimientos` proporciona seguridad por diseño:

1. **No existe endpoint DELETE ni UPDATE para movimientos** — si alguien borrara un movimiento, el historial quedaría corrupto. La API simplemente no expone esta operación.
2. **Las correcciones se hacen con contra-movimientos** — cada corrección queda en el historial con su nota obligatoria. Esto crea una auditoría automática: siempre se puede saber quién corrigió qué y por qué (aunque sin auth, no se sabe "quién").
3. **El `timestamp_creacion` lo genera el servidor** — no es editable por el cliente. Esto hace que la auditoría retroactiva sea confiable.

---

## Cifrado

**En tránsito**: HTTPS obligatorio. Los datos del inventario (nombres de productos, cantidades, notas de excepciones) viajan cifrados. Railway/Render proveen HTTPS automático.

**Contraseñas en reposo**: no aplica en v1 (sin autenticación). Cuando se implemente v2, usar **bcrypt con factor de trabajo ≥12**:

```python
import bcrypt
hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
```

**Datos del inventario en reposo**: las cantidades de stock e historial de movimientos no son datos sensibles al nivel de información médica o financiera. No se requiere cifrado adicional de la BD en v1.

---

## OWASP Top 10 relevantes

| OWASP | Vulnerabilidad | Mitigación específica |
|-------|---------------|----------------------|
| **A03: Injection (SQL)** | SQL injection en nombre de producto, nota de excepción, referencia de movimiento | ORM (SQLAlchemy) o queries parametrizadas — NUNCA concatenar input de usuario en SQL. Esto es crítico incluso sin autenticación |
| **A01: Broken Access Control** | Acceso a endpoints de modificación sin autenticación | En v1: restringir acceso a red interna. En v2: RBAC server-side verificado en cada request |
| **A08: Software and Data Integrity** | Manipulación del `timestamp_creacion` o del `flag_auditoria_retroactiva` | Estos campos se generan en el servidor y no son aceptados del cliente. El schema de Pydantic los ignora si el cliente los envía |
| **A05: Security Misconfiguration** | Servidor expuesto a internet sin auth | Deploy en red privada / VPN corporativa. No exponer el puerto directamente a internet |
| **A07: Identification and Authentication** | Sin autenticación (aceptado para v1) | Documentado como riesgo R-01. Mitigado con restricción de red. Pendiente para v2 |

---

## Principio de mínimo privilegio

Sin autenticación en v1, este principio aplica a nivel de base de datos:

| Capa | Aplicación |
|------|-----------|
| **BD** | Usuario de aplicación con permisos `SELECT, INSERT` sobre las tablas de negocio. Sin `UPDATE` ni `DELETE` en `movimientos` — esto hace técnicamente imposible que un bug en la aplicación borre el historial |
| **BD** | Sin permisos de superuser para el usuario de aplicación |
| **API** | El endpoint `POST /movimientos` no acepta `timestamp_creacion` del cliente — el servidor lo genera. Pydantic lo excluye del request schema |
| **API** | El endpoint de stock mínimo es `PATCH /productos/:id/stock-minimo` — actualiza solo ese campo, no el producto completo |

---

## Auditoría y trazabilidad

El ledger inmutable ya provee la mayoría de la trazabilidad necesaria:

| Acción | Cómo queda registrada |
|--------|----------------------|
| Registro de movimiento (entrada/salida) | `movimientos` con `timestamp_creacion` del servidor |
| Corrección de error | Contra-movimiento con nota obligatoria + referencia al movimiento original |
| Override (despacho en rojo) | `flag_excepcion=true` + `nota_excepcion` obligatoria en el movimiento |
| Movimiento retroactivo | `flag_auditoria_retroactiva=true` + `timestamp_creacion` real vs `fecha_hora_evento` declarada |
| Cambio de stock mínimo | Requiere log explícito — **no está en el schema actual**. Recomendación: añadir tabla `audit_stock_minimo` para registrar historial de cambios de este campo crítico |

La única brecha de auditoría en v1: sin autenticación, no se sabe *quién* registró cada movimiento. Esto es el riesgo R-01 aceptado explícitamente.
