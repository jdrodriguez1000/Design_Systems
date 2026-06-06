# Arquitectura del Sistema — Contador de Calorías

## Arquitectura recomendada: SPA Frontend-Only (Monolito de Cliente)

Este sistema es una **Single-Page Application (SPA) puramente de frontend**, sin backend propio. Toda la lógica reside en el navegador del usuario.

---

## Justificación

El requerimiento más determinante es la restricción EX-04: **un único usuario, sin autenticación**. Cuando no hay múltiples usuarios, no hay necesidad de servidor para gestionar sesiones, ni de base de datos centralizada, ni de APIs. El dato más sensible del sistema (el historial del día) puede vivir completamente en el navegador mediante `localStorage`.

Los criterios de decisión fueron:

| Criterio | Peso | Cómo lo satisface esta arquitectura |
|----------|------|-------------------------------------|
| Un solo usuario, sin auth | crítico | Sin backend, el aislamiento es natural: cada navegador es su propio mundo |
| Persistencia ante cierre de navegador (OV-05) | alto | `localStorage` es exactamente ese mecanismo |
| Reinicio automático a medianoche sin servidor (RN-20) | alto | Un `setTimeout` o chequeo en carga resuelve esto en el cliente |
| Simplicidad de despliegue | medio | Archivos estáticos: sin servidor que mantener |

---

## Alternativas consideradas

### Opción A: SPA + Backend (Node.js/Python + DB)
**¿Por qué se descarta?**
Añade complejidad sin resolver ningún problema real del proyecto. Si hay un único usuario que no necesita sincronizar entre dispositivos (EX-06), un servidor solo agrega: costos de hosting, latencia de red, gestión de autenticación, y un punto más de fallo. Es el antipatrón de *sobrediseño para uso personal*.

### Opción B: Aplicación web progresiva (PWA) con Service Worker
**¿Por qué no ahora?**
El funcionamiento offline fue explícitamente excluido (EX-05). Una PWA sería la evolución natural si Sofía algún día necesita usar la app sin conexión (DF-04), pero no es necesario hoy. Añadirla sin necesidad viola el principio de simplicidad.

### Opción C: App móvil nativa (React Native / Flutter)
**¿Por qué se descarta?**
El acceso principal es desde celular, pero como app web. No se mencionó necesidad de notificaciones push, acceso a hardware del dispositivo, ni distribución por tienda de apps. Una web funciona en celular sin fricción de instalación.

---

## Componentes principales

```
┌─────────────────────────────────────────────────────────┐
│                    NAVEGADOR (Browser)                  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │               UI Layer (HTML/CSS)                │   │
│  │  - Pantalla principal (total, saldo/exceso)      │   │
│  │  - Formulario de entrada de comida               │   │
│  │  - Historial del día (lista con botón eliminar)  │   │
│  │  - Campo de meta calórica                        │   │
│  └──────────────────┬──────────────────────────────┘   │
│                     │ eventos (click, input)            │
│  ┌──────────────────▼──────────────────────────────┐   │
│  │           Logic Layer (JavaScript)               │   │
│  │  - Validación de entradas (calorías > 0)         │   │
│  │  - Cálculo: total consumido                      │   │
│  │  - Cálculo: saldo restante / exceso calórico     │   │
│  │  - Lógica de reinicio diario (chequeo fecha)     │   │
│  └──────────────────┬──────────────────────────────┘   │
│                     │ read/write                        │
│  ┌──────────────────▼──────────────────────────────┐   │
│  │         Storage Layer (localStorage)             │   │
│  │  - Entradas de comida del día actual             │   │
│  │  - Meta calórica                                 │   │
│  │  - Fecha del día actual (para detectar reset)    │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**UI Layer**: Lo que ve Sofía. Recibe eventos del usuario y renderiza el estado.

**Logic Layer**: El cerebro. Valida, calcula y decide qué mostrar. No sabe nada de `localStorage` ni de DOM directamente.

**Storage Layer**: Abstracción sobre `localStorage`. El resto del sistema no habla con `localStorage` directamente — habla con esta capa.

---

## Trade-offs

| Ganamos | Sacrificamos |
|---------|-------------|
| Despliegue como archivos estáticos (GitHub Pages, Netlify — gratis) | Sin sincronización entre dispositivos (DF-04 requeriría backend) |
| Cero dependencia de conectividad para funcionar (irónicamente, pese a EX-05) | Sin historial en caso de limpiar caché del navegador |
| Sin costo de servidor ni base de datos | La lógica de reinicio depende de que la app se abra cerca de medianoche |
| Desarrollo rápido: todo en un solo contexto de ejecución | Difícil escalar a multi-usuario sin reescribir la arquitectura |

---

## Ejemplos reales

- **Google Keep (versión offline)**: guarda notas en `localStorage`/IndexedDB antes de sincronizar con el servidor. Mismo patrón: la fuente primaria de verdad es el cliente.
- **Excalidraw**: aplicación de diagramas SPA que funciona 100% offline con todo en memoria/localStorage.
- **TodoMVC**: el ejemplo canónico de esta arquitectura para apps de lista — exactamente el mismo patrón aplicado a un dominio diferente.
