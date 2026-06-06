# Protocolos de Comunicación — Contador de Calorías

## Situación especial: sistema sin comunicación en red

Este sistema es una SPA frontend-only. **No hay comunicación entre cliente y servidor** porque no hay servidor. Esta es la decisión más radical de la arquitectura — y también la más correcta para el problema planteado.

La "comunicación" en este sistema ocurre en dos dimensiones:
1. **Interna (en-proceso)**: entre los módulos de la aplicación dentro del navegador
2. **Con el almacenamiento**: entre la lógica de la app y `localStorage`

---

## Comunicación interna (módulos del navegador)

La comunicación entre el Logic Layer y el UI Layer sigue el patrón de **llamadas de función directas** — la forma más simple y eficiente cuando todo corre en el mismo proceso JavaScript.

```
UI Layer ──[evento DOM]──▶ Logic Layer ──[operación]──▶ Storage Layer
UI Layer ◀──[render()]────────────────────────────────── Logic Layer
```

No hay overhead de serialización, no hay latencia de red, no hay necesidad de protocolos. El "protocolo" es la firma de las funciones JavaScript.

---

## ¿REST, GraphQL o gRPC? — Análisis para la fase futura (DF-04)

Aunque hoy no aplica, documentar este análisis es valioso para cuando se implemente sincronización entre dispositivos.

### REST API — Recomendado si se añade backend

**¿Por qué REST sobre GraphQL para este dominio?**

GraphQL tiene su valor cuando hay muchos tipos de consultas diferentes sobre grafos de datos complejos (piensa en GitHub, Shopify). Para este sistema, las operaciones son:
- `GET /entries` — obtener historial
- `POST /entries` — agregar entrada
- `DELETE /entries/:id` — eliminar entrada
- `GET /goal` — obtener meta
- `PUT /goal` — actualizar meta

5 endpoints. REST es perfectamente adecuado. GraphQL añadiría un schema, un resolver, y complejidad de operaciones que no aportan valor para 5 operaciones CRUD simples.

**¿Por qué REST sobre gRPC?**

gRPC es excelente para comunicación entre microservicios internos de alta frecuencia (ej: servicios de Uber comunicándose entre sí miles de veces por segundo). Para una app personal con acceso desde un navegador web, gRPC requiere un proxy HTTP adicional y no está soportado nativamente en browsers. El overhead no tiene justificación.

---

## Comunicación en tiempo real — No aplica hoy

WebSockets y SSE (Server-Sent Events) sirven para sincronización en tiempo real entre múltiples clientes (ej: Google Docs actualizando mientras múltiples usuarios editan). Con un único usuario en un único dispositivo, esto no tiene sentido. Si DF-04 se implementa algún día, SSE podría ser útil para notificar al cliente web cuando se registra una entrada desde el celular y el navegador de escritorio está abierto simultáneamente.

---

## Formato de datos para localStorage

**JSON** — la única opción práctica para `localStorage`.

`localStorage` solo almacena strings. JSON es el formato nativo de JavaScript para serializar objetos a string. No hay ventaja en usar otro formato:

- **Protocol Buffers (protobuf)**: eficiente en tamaño y velocidad de parsing. Relevante para comunicación de alta frecuencia en red. Para ~50 entradas al día en localStorage, el overhead de JSON es invisible.
- **Avro**: diseñado para streaming de datos (Kafka, Hadoop). No aplica.
- **CSV**: más compacto para datos tabulares simples, pero más frágil para estructuras con campos opcionales.

---

## Diseño de la capa de comunicación interna

Aunque no hay red, sí hay un contrato implícito entre módulos. Documentarlo evita acoplamiento accidental:

```javascript
// Contrato del StorageRepository (equivalente a una API interna)
interface StorageRepository {
  getEntries(): FoodEntry[]          // Nunca retorna null — retorna array vacío
  saveEntries(entries: FoodEntry[]): void
  getGoal(): number | null           // null = sin meta configurada
  saveGoal(goal: number | null): void
  getLastDate(): string | null       // formato YYYY-MM-DD
  saveDate(date: string): void
}

// Contrato de FoodEntry
interface FoodEntry {
  id: number      // timestamp como identificador
  nombre: string  // texto libre, no vacío
  calorias: number // entero > 0
}
```

Este "contrato" es exactamente lo que en sistemas con red llamaríamos un **schema de API** o un **data contract**.
