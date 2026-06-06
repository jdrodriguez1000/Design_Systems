# Stack Tecnológico — Contador de Calorías

## Stack recomendado

Este sistema tiene el stack más simple posible que resuelve el problema. Cada tecnología está justificada por necesidad, no por tendencia.

---

## Por capa

### Frontend / Cliente (la única capa)

| Opción | Recomendación | Justificación |
|--------|--------------|---------------|
| **Vanilla HTML + CSS + JavaScript** | ✅ Recomendado para V1 | Sin build step, sin dependencias, máxima transparencia educativa |
| **Vite + Vanilla JS** | ✅ Recomendado si se quiere bundling | Permite módulos ES, tree-shaking, y `npm test` con Vitest |
| **React** | Válido si el objetivo es practicar React | Añade reactividad automática a costo de complejidad de setup |
| **Vue 3** | Válido si el objetivo es practicar Vue | Composition API es más simple que React para este caso de uso |
| **Svelte** | Válido, mínimo overhead en runtime | El template compila a JS puro — el bundle más pequeño |

**Decisión recomendada**: `Vite + Vanilla JS + TypeScript opcional`. Vite da un servidor de desarrollo con hot-reload, soporte para módulos ES (que permiten separar el código en archivos), y Vitest para tests — todo sin la complejidad de Webpack.

**¿Por qué no React para V1?**: React resuelve el problema de actualización eficiente del DOM en aplicaciones con muchos componentes que cambian frecuentemente. Para una pantalla con 5-7 elementos, el DOM manipulation manual con `textContent` y `appendChild` es perfectamente mantenible.

---

### Backend / API — No aplica en V1

No existe. Ver `005_architecture.md` para la justificación.

**Si se implementa DF-04**: Node.js + Express (o Fastify para mejor rendimiento) sería el stack más natural si ya se está usando JavaScript en el frontend. Python + FastAPI sería igualmente válido.

---

### Base de datos principal — localStorage

Ya documentado en `010_database_model.md`. `localStorage` es la única base de datos necesaria.

**Si se implementa DF-04**: PostgreSQL. Es la base de datos relacional más robusta del ecosistema open-source, con soporte para JSON nativo (por si se quiere guardar el array de entradas como JSONB) y excelente soporte en todos los proveedores cloud.

---

### Base de datos secundaria (caché, búsqueda) — No aplica

Sin backend, no hay Redis ni Elasticsearch. En el futuro, si el historial de días anteriores (DF-02) se implementa, un caché de Redis para el día actual tendría sentido.

---

### Infraestructura y cloud

| Propósito | Tecnología | Justificación |
|-----------|-----------|---------------|
| **Hosting (V1)** | GitHub Pages o Netlify | Gratis para sitios estáticos, HTTPS automático, deploy por push |
| **Dominio personalizado** | Proveedor de dominio cualquiera (Namecheap, Google Domains) | Opcional — `sofiacalorias.netlify.app` funciona sin dominio propio |

**Si se añade backend**: Railway, Render o Fly.io ofrecen free tiers para proyectos personales. Evitar AWS/GCP/Azure para un proyecto personal — la complejidad y el costo no se justifican.

---

### Herramientas de testing

| Herramienta | Propósito | Alternativa |
|-------------|----------|------------|
| **Vitest** | Unit tests + coverage | Jest (más conocido, más pesado) |
| **Playwright** | E2E tests opcionales | Cypress |
| `jest-localstorage-mock` | Mock de localStorage en Node.js | Implementación manual |

---

### Herramientas de observabilidad

| Herramienta | Propósito | Cuándo activar |
|-------------|----------|---------------|
| **DevTools del navegador** | Debugging y verificación de localStorage | Siempre disponible |
| **Sentry (free tier)** | Alertas de errores JavaScript en producción | Si se quiere saber cuándo la app falla |
| Sin métricas | No hay backend que monitorear | — |

---

## Compatibilidad del stack

```
Vite (build tool)
  └── sirve: HTML, CSS, JavaScript (modules)
       └── usa: localStorage (browser API nativa)
            └── tests: Vitest (mismo runtime que Vite)
                 └── CI/CD: GitHub Actions
                      └── deploy: GitHub Pages / Netlify
```

Todo es JavaScript/Node.js — sin fricción de interoperabilidad entre lenguajes.

---

## Alternativas descartadas con razón específica

| Tecnología | Por qué se descartó |
|-----------|---------------------|
| **React** para V1 | Overhead de setup (Babel, JSX, hooks) no justificado para una pantalla |
| **Next.js** | Framework para aplicaciones con SSR/SSG y múltiples rutas; overkill para una SPA de una pantalla |
| **Firebase** | Persistencia en la nube con auth integrado — correcto si se necesitara sync entre dispositivos, pero EX-06 lo excluye. Añade dependencia de Google y vendor lock-in |
| **Electron** | Aplicación de escritorio nativa — Sofía accede principalmente desde celular; una app nativa de escritorio no mejora su experiencia |
| **SQLite (browser via WASM)** | Disponible via `sql.js` — fascinante tecnológicamente, completamente innecesario para este problema |

---

## Curva de aprendizaje y madurez

| Tecnología | Curva | Madurez | Ecosistema |
|-----------|-------|---------|-----------|
| Vanilla JS | Baja (si se conoce JS) | Alta — el lenguaje no cambia | Estable |
| Vite | Baja | Alta — v5 estable | Activo, bien mantenido |
| Vitest | Baja (similar a Jest) | Media-Alta — v2 estable | Activo |
| localStorage API | Mínima | Alta — web estándar desde 2009 | Sin cambios desde entonces |

**Conclusión**: el stack completo puede dominarse en un fin de semana para alguien con conocimiento básico de HTML/CSS/JS. Esa es exactamente la característica correcta para un sistema de esta complejidad.
