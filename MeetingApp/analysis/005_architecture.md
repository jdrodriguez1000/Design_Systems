# Arquitectura del Sistema — MeetingApp

## Arquitectura recomendada: Monolito por capas (Layered Monolith)

Un monolito bien estructurado con separación clara de capas: presentación, lógica de negocio y datos.

---

## Justificación

El criterio principal es la escala del problema: **25 usuarios, 3 salas, una empresa**. La lección más importante de diseño de sistemas es que la arquitectura debe ser proporcional al problema. Los microservicios, el event-sourcing y la arquitectura hexagonal son soluciones reales — pero para problemas de escala real.

| Criterio | Cómo lo satisface un monolito |
|----------|-------------------------------|
| 25 usuarios concurrentes máximo | Un solo servidor con 1-2 CPUs maneja esto sin problema |
| Lógica de conflicto transaccional | En un monolito, toda la lógica de reserva ocurre en una sola transacción de DB |
| Sin equipo de ingeniería dedicado | Un monolito es operable por una persona; los microservicios no |
| Tiempo de desarrollo corto | Sin overhead de comunicación entre servicios, service discovery, ni orquestación |

El segundo factor decisivo: **la lógica de resolución de conflictos necesita atomicidad**. Cuando una reserva prioritaria desplaza a una estándar, estas cosas deben ocurrir en la misma transacción de base de datos:
1. Cancelar la reserva existente
2. Crear la nueva reserva
3. Registrar el desplazamiento
4. Encolar la notificación al afectado

En un monolito con una BD relacional, esto es un `BEGIN/COMMIT`. En microservicios, requiere el patrón Saga con compensación — complejidad de orden de magnitud superior sin ningún beneficio para este tamaño.

*Como se aprendió en Contador_Calorias con YAGNI: no se implementa una capacidad porque podría necesitarse en el futuro. Los microservicios se justifican cuando hay equipos independientes desplegando componentes independientemente, o cuando la escala supera lo que un único proceso puede manejar. Ninguna de las dos condiciones existe aquí.*

---

## Alternativas consideradas

### Opción A: Microservicios
- **¿Cuándo sería correcto?**: cuando la empresa tenga 200+ empleados y múltiples edificios, con equipos independientes trabajando en diferentes partes del sistema.
- **¿Por qué se descarta ahora?**: los microservicios introducen latencia de red entre servicios, necesidad de service mesh, distributed tracing, y el problema de consistencia eventual en transacciones que aquí necesitan ser ACID. Para 25 usuarios es un cañón matando una mosca.

### Opción B: Serverless (AWS Lambda / Vercel Functions)
- **¿Cuándo sería correcto?**: aplicaciones con tráfico muy variable (0 a miles en segundos), o como backend de un MVP sin servidor dedicado.
- **¿Por qué se descarta?**: el cold start de funciones serverless introduce latencia impredecible. La lógica de reserva con transacciones de DB y locking pesimista no se lleva bien con conexiones efímeras de Lambda. Y para 25 usuarios el costo/complejidad no se justifica.

### Opción C: SPA con backend BaaS (Firebase, Supabase)
- **¿Cuándo sería correcto?**: MVPs rápidos donde no se quiere escribir backend propio.
- **¿Por qué se descarta?**: la lógica de desplazamiento automático con transacciones atómicas requiere control total sobre las operaciones de DB. Firebase/Supabase tienen reglas de seguridad pero no permiten lógica transaccional compleja del lado del servidor sin Cloud Functions — lo cual se convierte en un mini-backend de todas formas.

---

## Componentes principales

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENTE (Browser)                          │
│  React / Vue SPA — UI de consulta, reserva, notificaciones     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS / REST API
┌──────────────────────────▼──────────────────────────────────────┐
│                      BACKEND MONOLITO                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Capa de Presentación (API Layer)                        │   │
│  │  - Controladores REST                                    │   │
│  │  - Autenticación / autorización (middleware)             │   │
│  │  - Serialización de respuestas                           │   │
│  └──────────────────────┬──────────────────────────────────┘   │
│                         │                                       │
│  ┌──────────────────────▼──────────────────────────────────┐   │
│  │  Capa de Negocio (Domain/Service Layer)                  │   │
│  │  - ReservationService: crear, cancelar, desplazar        │   │
│  │  - ConflictResolver: detectar y resolver solapamientos   │   │
│  │  - PriorityEngine: clasificar y comparar prioridades     │   │
│  │  - NotificationService: encolar notificaciones in-app    │   │
│  └──────────────────────┬──────────────────────────────────┘   │
│                         │                                       │
│  ┌──────────────────────▼──────────────────────────────────┐   │
│  │  Capa de Datos (Repository Layer)                        │   │
│  │  - ReservationRepository                                 │   │
│  │  - RoomRepository                                        │   │
│  │  - UserRepository                                        │   │
│  │  - NotificationRepository                                │   │
│  └──────────────────────┬──────────────────────────────────┘   │
└──────────────────────────┼──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    PostgreSQL                                    │
│  reservas | salas | usuarios | notificaciones | desplazamientos │
└─────────────────────────────────────────────────────────────────┘
```

---

## Trade-offs

| Ganamos | Sacrificamos |
|---------|-------------|
| Transacciones ACID para la lógica de conflictos | Escalabilidad independiente de componentes |
| Un solo deployment, un solo proceso que monitorear | Si el proceso cae, todo cae (mitigable con PM2/Docker restart) |
| Debugging simple: un stack trace, un log | Si la app crece mucho, el monolito puede volverse difícil de mantener |
| Latencia mínima (sin comunicación entre servicios) | Cambios en cualquier capa requieren redesplegar todo |

---

## Ejemplos reales

- **Stack Overflow** (hasta ~2020): todo un monolito en ASP.NET manejando millones de usuarios. Escalabilidad vertical + caching agresivo.
- **GitHub** (primeros 10 años): Ruby on Rails monolito. Empezaron a extraer microservicios solo cuando la escala lo exigió — no antes.
- **Basecamp**: sigue siendo un monolito deliberado. DHH y Jason Fried lo defienden públicamente como la decisión correcta para su escala.
- **Shopify**: empezó como un monolito Rails. Lo llaman "The Modular Monolith" — misma idea que aquí: modularidad interna sin separación de procesos.
