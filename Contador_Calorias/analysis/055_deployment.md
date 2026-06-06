# Estrategia de Despliegue — Contador de Calorías

## Situación especial: archivos estáticos

Una SPA frontend-only **no necesita servidor de aplicación**. Solo necesita un servidor de archivos estáticos (o ni eso, si Sofía abre el `index.html` directamente desde su disco). Esto simplifica radicalmente el despliegue.

---

## Estrategia recomendada: Despliegue estático en CDN

**Netlify, Vercel o GitHub Pages** — cualquiera de los tres.

| Opción | Costo | Ventaja principal |
|--------|-------|------------------|
| **GitHub Pages** | Gratis | Integrado con el repositorio; un push y está desplegado |
| **Netlify** | Gratis (plan free generoso) | HTTPS automático, CDN global, deploy preview por PR |
| **Vercel** | Gratis (plan hobby) | Mismo perfil que Netlify |
| Archivo local | Cero | Sin dependencia de internet — Sofía abre `index.html` desde el disco |

**Recomendación concreta**: GitHub Pages si ya se usa Git para el código. La app queda en `https://username.github.io/contador-calorias/`.

---

## Estrategia de despliegue: No aplica blue/green ni canary

Blue/green y canary releases resuelven el problema de desplegar a muchos usuarios sin interrumpir el servicio. Con una sola usuaria y archivos estáticos, el "despliegue" es simplemente subir los archivos nuevos. Si algo falla, `git revert` + `git push` lo deshace en minutos.

**¿Por qué no blue/green?**: esta estrategia requiere dos entornos activos simultáneamente y un load balancer que redirija tráfico. Para un archivo estático y una usuaria, añade complejidad sin resolver ningún problema real.

---

## Containerización — No aplica

Docker es valioso cuando el entorno de ejecución tiene dependencias (Node.js version, librerías de sistema, variables de entorno). Una app de HTML/CSS/JS puro no tiene dependencias de entorno. Poner un `index.html` en un contenedor Docker es exceso de ingeniería para este caso.

**Si se añade backend (DF-04)**: Docker se convierte en una herramienta valiosa para garantizar que el entorno de desarrollo sea idéntico al de producción.

---

## Pipeline CI/CD recomendado

Para este sistema, un pipeline mínimo es suficiente y correcto:

```yaml
# .github/workflows/deploy.yml (GitHub Actions)

on:
  push:
    branches: [main]

jobs:
  test-and-deploy:
    steps:
      - Checkout código
      - Instalar dependencias (npm install)
      - Ejecutar tests unitarios (npm test)
      - Si los tests pasan → Deploy a GitHub Pages
      - Si los tests fallan → No deployar, notificar
```

**Etapas en orden**:
1. `lint`: verificar formato de código (opcional pero útil)
2. `test`: tests unitarios e integración
3. `build`: si se usa un bundler (Vite, Webpack) — empaquetar
4. `deploy`: subir a hosting estático

**Condición de seguridad**: nunca deployar si los tests fallan. Esto es automático en GitHub Actions con la configuración por defecto.

---

## Ambientes necesarios

| Ambiente | Para qué | Cómo |
|----------|---------|------|
| **Local** | Desarrollo y testing | `index.html` abierto directamente, o `npx serve .` |
| **Producción** | Sofía usa la app | GitHub Pages / Netlify |

**¿Por qué no un ambiente de staging?** Staging tiene valor cuando hay que validar un cambio con usuarios reales antes de afectar a todos los usuarios. Con una única usuaria que es también la desarrolladora (o la persona que aprueba los cambios), staging no aporta valor. El ambiente local cumple esa función.

---

## Rollback

El rollback de un archivo estático es inmediato:

```bash
git revert HEAD  # crea un commit que deshace el último cambio
git push         # el CI/CD lo detecta y deploy la versión anterior
```

Tiempo de rollback estimado: 2-3 minutos (tiempo del pipeline).

---

## Observabilidad en producción

Para una app personal sin backend, la "observabilidad" es mínima:

| Qué monitorear | Cómo |
|----------------|------|
| ¿La app carga correctamente? | Verificación manual periódica |
| ¿`localStorage` se está llenando? | Verificar en DevTools → Application → Storage |
| Errores JavaScript | Habilitar Sentry (nivel free) si se quiere alertas automáticas |

**Logging**: para una app local, `console.error` para errores inesperados es suficiente. No se necesita un sistema de logging centralizado.

**Métricas**: no aplica sin backend. Si se añadiera backend, las métricas mínimas serían: latencia de API y tasa de error.

---

## Infraestructura como código (IaC)

No aplica para archivos estáticos en GitHub Pages/Netlify. La "infraestructura" es el archivo `netlify.toml` o la configuración en el panel web del hosting — trivial de mantener manualmente.

Si se añadiera backend (DF-04), Terraform o Pulumi serían las herramientas adecuadas para provisionar la infraestructura cloud.
