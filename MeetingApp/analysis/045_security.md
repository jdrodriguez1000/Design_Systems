# Seguridad — MeetingApp

## Modelo de amenazas

A diferencia de Contador_Calorias (app personal sin autenticación), este sistema tiene múltiples usuarios con roles y datos de empresa. El modelo de amenazas es más rico.

| Actor malicioso | Vector de ataque | Probabilidad | Impacto |
|----------------|-----------------|-------------|---------|
| Empleado intentando reservar por otro | Usar el token JWT de un colega | Baja (acceso físico o phishing) | Medio — reserva a nombre falso |
| Empleado intentando marcar como urgente sin ser gerente | Manipular el request POST directamente (bypass de UI) | Media (cualquiera con DevTools) | Alto — eleva prioridad sin autorización |
| Empleado intentando crear usuarios sin ser admin | Llamar `POST /usuarios` con token propio | Media | Alto — crea cuentas no autorizadas |
| SQL Injection a través de inputs de formulario | Inyectar SQL en nombre de reunión o usuario | Media si no se usan queries parametrizadas | Alto — acceso total a la BD |
| Ataque de fuerza bruta al login | Iterar contraseñas hasta encontrar la correcta | Alta (automatable) | Alto — acceso a cualquier cuenta |
| IDOR (Insecure Direct Object Reference) | Acceder a `/reservas/UUID_ajeno` para ver/cancelar | Media | Medio — ver reservas de otros |

---

## Autenticación: JWT con refresh tokens

**¿Por qué JWT y no sesiones?**

Para una aplicación con frontend separado (SPA + API backend), JWT es la elección más natural. Las sesiones server-side requieren almacenamiento de estado en el servidor (Redis o DB) — añade una dependencia más. JWT es stateless: el token contiene la información del usuario y el servidor la verifica criptográficamente sin consultar nada.

**Estructura del JWT**:
```json
{
  "sub": "uuid-del-usuario",
  "rol": "gerente",
  "nombre_usuario": "laura.jimenez",
  "exp": 1717689600,
  "iat": 1717686000
}
```

**Tiempo de vida**:
- Access token: 15-60 minutos (corto — si se roba, expira rápido)
- Refresh token: 7 días (almacenado en httpOnly cookie — no accesible desde JavaScript)

**¿Por qué incluir el rol en el JWT?** Para que el backend pueda verificar permisos sin consultar la BD en cada request. El trade-off: si se cambia el rol de un usuario, el cambio no es efectivo hasta que expire el access token. Para este caso de uso (25 usuarios, cambios de rol infrecuentes), es aceptable.

---

## Autorización: RBAC verificado en el servidor

**La regla más crítica**: la verificación de permisos SIEMPRE ocurre en el backend. La UI puede ocultar botones por rol, pero esto es solo UX — un atacante puede hacer el request directamente ignorando la UI.

```python
# Middleware que verifica el JWT y extrae el usuario
# Se aplica a TODOS los endpoints excepto /auth/login
@app.before_request
def require_auth():
    token = request.headers.get('Authorization', '').replace('Bearer ', '')
    request.user = jwt.decode(token, SECRET_KEY)

# Decorador RBAC en cada endpoint sensible
@require_permission('marcar_urgente')  # Verifica server-side
@app.route('/reservas/<id>/urgente', methods=['PATCH'])
def mark_urgent(id):
    # Si llega aquí, el usuario tiene el permiso — verificado en el servidor
    ...
```

**ACP-16**: "Solo el rol gerente puede marcar una reunión como urgente" — esta validación NUNCA puede depender de lo que el frontend envíe. El backend verifica el `rol` del JWT en cada request.

---

## Cifrado

**En tránsito**: HTTPS obligatorio. Los tokens JWT, contraseñas (durante el login), y datos de reservas viajan cifrados. Si el hosting es Netlify/Vercel para el frontend o Railway/Render para el backend, HTTPS está activado por defecto.

**Contraseñas en reposo**: nunca almacenar contraseñas en texto claro. Usar **bcrypt** con factor de trabajo ≥12:

```python
import bcrypt

# Al crear usuario
hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))

# Al verificar login
valid = bcrypt.checkpw(input_password.encode(), stored_hash)
```

**Datos de reservas en reposo**: no se requiere cifrado adicional. Los nombres de reuniones y horarios no son datos sensibles al nivel de información médica o financiera.

---

## OWASP Top 10 relevantes

| OWASP | Vulnerabilidad | Mitigación específica |
|-------|---------------|----------------------|
| **A01: Broken Access Control** | Empleado accede a reserva/endpoint que no le pertenece | RBAC server-side + verificar `reserva.usuario_id == request.user.id` antes de modificar |
| **A03: Injection (SQL)** | Inyección SQL en queries de disponibilidad | ORM o queries parametrizadas — NUNCA concatenar input de usuario en SQL |
| **A07: Authentication Failures** | Fuerza bruta al login | Rate limiting en `/auth/login` (máx 5 intentos/min por IP), lockout temporal |
| **A04: Insecure Design** | Permitir que el cliente decida la prioridad de su reserva | La prioridad se calcula en el servidor basada en `tipo_reunion` + `rol` — no es un campo que el cliente envíe |
| **A05: Security Misconfiguration** | JWT con algoritmo "none", secreto débil | `alg: HS256` explícito, secreto de ≥256 bits generado aleatoriamente |

---

## Principio de mínimo privilegio

| Capa | Aplicación |
|------|-----------|
| **BD** | Usuario de aplicación con permisos solo sobre las tablas del sistema (no superuser, no acceso a tablas del sistema) |
| **API** | Cada endpoint verifica exactamente el permiso necesario — no "está autenticado" sino "tiene permiso X" |
| **Notificaciones** | Un usuario solo puede leer sus propias notificaciones — `WHERE destinatario_id = $usuario_id` siempre |
| **Retroactivas** | La verificación `es_retroactiva` está en el server, no en el cliente — el cliente no puede forzar `es_retroactiva: true` |

---

## Auditoría y trazabilidad

Acciones a registrar obligatoriamente:

| Acción | Por qué registrar |
|--------|-------------------|
| Login exitoso y fallido | Detectar fuerza bruta; reconstruir incidentes de acceso |
| Creación/cancelación de reserva | Registro completo del historial de ocupación |
| Desplazamiento ejecutado | Trazabilidad del por qué una reserva fue cancelada (tabla `desplazamientos` ya la provee) |
| Creación/modificación de usuario | Detectar creación de cuentas no autorizadas |
| Marcado manual como urgente | Auditoría del uso de la capacidad exclusiva del gerente |

```sql
-- Tabla de auditoría simple
CREATE TABLE audit_log (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    usuario_id  UUID REFERENCES usuarios(id),
    accion      TEXT NOT NULL,
    entidad     TEXT NOT NULL,
    entidad_id  UUID,
    ip          TEXT,
    timestamp   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata    JSONB
);
```

*A diferencia de Contador_Calorias (sin backend, sin logs necesarios), aquí el log de auditoría es crítico para reconstruir "¿por qué desapareció la reserva de X?" ante cualquier disputa.*
