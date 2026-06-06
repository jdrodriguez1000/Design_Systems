Analiza los requerimientos del proyecto y genera documentos educativos de diseño de sistemas.

El nombre del proyecto es: $ARGUMENTS

## Instrucciones

Lee los siguientes 8 archivos de requerimientos del proyecto:

- `$ARGUMENTS/requirements/shared_understanding.md`
- `$ARGUMENTS/requirements/domain_glossary.md`
- `$ARGUMENTS/requirements/scope_boundaries.md`
- `$ARGUMENTS/requirements/failure_behavior.md`
- `$ARGUMENTS/requirements/acceptance_criteria.md`
- `$ARGUMENTS/requirements/bdd_features.md`
- `$ARGUMENTS/requirements/data_contracts.md`
- `$ARGUMENTS/requirements/error_exception_policy.md`

Antes de generar los documentos, consulta el historial de lecciones aprendidas:

```bash
python -c "
import sqlite3
from pathlib import Path

if not Path('lessons.db').exists():
    print('SIN_HISTORIAL')
else:
    conn = sqlite3.connect('lessons.db')
    total = conn.execute('SELECT COUNT(*) FROM lessons').fetchone()[0]
    if total == 0:
        print('SIN_HISTORIAL')
    else:
        rows = conn.execute('''
            SELECT project, title, context, learned, why_matters, tags
            FROM lessons
            ORDER BY created_at DESC
        ''').fetchall()
        conn.close()
        for r in rows:
            print(f'PROYECTO={r[0]} | TITULO={r[1]} | CONTEXTO={r[2]} | APRENDIDO={r[3]} | IMPORTA={r[4]} | TAGS={r[5]}')
"
```

Si el resultado es `SIN_HISTORIAL`, continúa normalmente sin contexto previo.

Si hay lecciones, tenlas presentes al generar cada documento:
- Si una lección previa es **directamente aplicable** al sistema actual, referenciarla explícitamente: *"Como se aprendió en [Proyecto], [concepto] aplica aquí porque..."*
- Si una lección previa fue aprendida en un contexto diferente pero el patrón es similar, mencionarla como advertencia o consideración
- No forzar referencias cuando no apliquen — solo cuando aporten valor real al análisis

Con base en el contenido completo de los requerimientos y el historial de lecciones, genera los siguientes documentos en la carpeta `$ARGUMENTS/analysis/`. Cada documento debe ser **educativo**: no solo indicar QUÉ se recomienda sino explicar el POR QUÉ, las alternativas consideradas, los trade-offs y referencias a sistemas reales cuando sea relevante. Escribe en español.

---

### 1. `$ARGUMENTS/analysis/005_architecture.md`

Título: Arquitectura del Sistema

Incluye:
- **Arquitectura recomendada**: estilo arquitectónico (monolítico, microservicios, serverless, event-driven, hexagonal, etc.)
- **Justificación**: por qué esta arquitectura y no otra — criterios de decisión basados en los requerimientos
- **Alternativas consideradas**: otras arquitecturas que podrían aplicar y por qué se descartan
- **Componentes principales**: lista y descripción de cada componente clave
- **Diagrama conceptual**: representación en texto/ASCII del sistema
- **Trade-offs**: qué ganamos y qué sacrificamos con esta decisión
- **Ejemplos reales**: empresas o sistemas conocidos que usan esta arquitectura (Netflix, Uber, Amazon, etc.)

---

### 2. `$ARGUMENTS/analysis/010_database_model.md`

Título: Modelo de Base de Datos

Incluye:
- **Tipo de base de datos recomendada**: SQL, NoSQL (documento, clave-valor, grafo, columnar), NewSQL, o combinación — con justificación
- **Por qué este modelo**: relación con los patrones de acceso a datos del proyecto
- **Esquema principal**: entidades, relaciones y campos clave
- **Estrategia de indexación**: qué índices crear y por qué
- **Alternativas**: otras bases de datos que podrían funcionar y sus trade-offs
- **Conceptos aplicados**: ACID, BASE, normalización/desnormalización según corresponda — explicados en contexto

---

### 3. `$ARGUMENTS/analysis/015_design_patterns.md`

Título: Patrones de Diseño

Incluye:
- **Patrones creacionales aplicables**: Factory, Builder, Singleton, etc. — solo los relevantes para este proyecto con justificación
- **Patrones estructurales aplicables**: Adapter, Facade, Decorator, etc. — solo los relevantes
- **Patrones de comportamiento aplicables**: Observer, Strategy, Command, etc. — solo los relevantes
- **Patrones arquitectónicos**: CQRS, Event Sourcing, Saga, Repository, etc. — si aplican
- **Por qué cada patrón**: el problema concreto que resuelve en este contexto
- **Pseudocódigo conceptual**: estructura básica de los 2-3 patrones más importantes

---

### 4. `$ARGUMENTS/analysis/020_data_structures.md`

Título: Estructuras de Datos

Incluye:
- **Estructuras recomendadas**: cuáles son clave para este sistema y dónde se usan
- **Casos de uso concretos**: para cada estructura, dónde y cómo aparece en el dominio del proyecto
- **Complejidad relevante**: Big O de las operaciones críticas para este caso
- **Estructuras especializadas**: colas de prioridad, árboles, grafos, bloom filters, etc. si aplican
- **Justificación**: por qué estas estructuras y no otras

---

### 5. `$ARGUMENTS/analysis/025_communication_protocols.md`

Título: Protocolos de Comunicación

Incluye:
- **Comunicación sincrónica**: REST vs GraphQL vs gRPC — cuál recomendar y por qué para este proyecto
- **Comunicación asincrónica**: message queues, event streaming (Kafka, RabbitMQ, SQS) — si aplica y por qué
- **Comunicación en tiempo real**: WebSockets, SSE, Long Polling — si aplica y por qué
- **Formato de datos**: JSON, Protocol Buffers, Avro, etc. — cuál y por qué
- **Diseño de API**: principios clave para las APIs de este sistema
- **Comunicación entre servicios**: si hay múltiples componentes, cómo se comunican
- **Trade-offs**: latencia, throughput y complejidad de cada opción evaluada

---

### 6. `$ARGUMENTS/analysis/030_scalability.md`

Título: Estrategias de Escalabilidad

Incluye:
- **Escalabilidad horizontal vs vertical**: cuál aplica, cuándo y por qué
- **Estrategia de caché**: Redis, Memcached, CDN, caché en aplicación — qué usar, dónde y por qué
- **Load balancing**: estrategias y algoritmos recomendados para este caso
- **Database scaling**: replicación, sharding, particionamiento — si aplica
- **Cuellos de botella anticipados**: dónde fallará primero el sistema bajo carga alta
- **Estimaciones de capacidad**: cálculos aproximados basados en la escala esperada de los requerimientos
- **Auto-scaling**: cuándo y cómo escalar automáticamente

---

### 7. `$ARGUMENTS/analysis/035_advanced_concepts.md`

Título: Conceptos Avanzados

Incluye:
- **Teorema CAP**: cuáles dos propiedades prioriza este sistema (CP, AP o CA) y por qué — con explicación del teorema aplicada al contexto
- **Modelo de consistencia**: fuerte, eventual o causal — justificación según los requerimientos
- **Alta disponibilidad**: estrategias para garantizar uptime según los SLAs del proyecto
- **Tolerancia a fallos**: circuit breakers, retry policies, bulkhead pattern — cuáles aplican
- **ACID vs BASE**: cuál aplica a cada componente del sistema y por qué

---

### 8. `$ARGUMENTS/analysis/040_maintainability.md`

Título: Mantenibilidad

Incluye:
- **Principios SOLID aplicados**: cuáles de los cinco principios son más críticos para este sistema y por qué
- **Separación de concerns**: cómo dividir responsabilidades entre capas y módulos
- **Modularidad**: qué tan fácil es reemplazar o modificar cada parte del sistema de forma independiente
- **Deuda técnica anticipada**: qué atajos tomados hoy generarán problemas mañana y cómo mitigarlos
- **Estrategia de versionado**: cómo evolucionar el sistema sin romper lo que ya funciona
- **Documentación interna**: qué documentar en el código y qué no
- **Indicadores de mal diseño**: code smells específicos a vigilar en este tipo de sistema

---

### 9. `$ARGUMENTS/analysis/045_security.md`

Título: Seguridad

Incluye:
- **Modelo de amenazas**: qué actores maliciosos podrían atacar este sistema y cómo
- **Autenticación y autorización**: estrategia recomendada para este dominio (JWT, OAuth, sesiones, RBAC, etc.)
- **Cifrado**: qué datos cifrar en tránsito y en reposo, y con qué mecanismos
- **Vulnerabilidades OWASP relevantes**: las del Top 10 que aplican directamente a este sistema
- **Principio de mínimo privilegio**: cómo aplicarlo en cada capa
- **Seguridad en la base de datos**: inyección SQL, acceso, backups cifrados
- **Auditoría y trazabilidad**: qué acciones registrar para detectar y reconstruir incidentes

---

### 10. `$ARGUMENTS/analysis/050_testing_strategy.md`

Título: Estrategia de Testing

Incluye:
- **Pirámide de testing aplicada**: proporción recomendada de unit / integration / e2e para este sistema
- **Qué testear obligatoriamente**: los casos críticos que nunca deben quedar sin cobertura
- **Qué no vale la pena testear**: evitar over-testing que no aporta valor
- **Testing de la base de datos**: cómo probar migraciones, queries y transacciones
- **Testing de APIs**: contract testing, mocking de dependencias externas
- **Testing de rendimiento**: cuándo y cómo hacer load testing para este sistema
- **Estrategia de datos de prueba**: cómo generar y gestionar datos de test sin comprometer producción

---

### 11. `$ARGUMENTS/analysis/055_deployment.md`

Título: Estrategia de Despliegue

Incluye:
- **Estrategia de despliegue recomendada**: blue/green, canary, rolling update — cuál aplica y por qué
- **Containerización**: si aplica Docker/Kubernetes para este sistema y en qué escenario
- **Pipeline CI/CD**: etapas recomendadas para este tipo de proyecto
- **Ambientes**: cuántos ambientes necesita este sistema (dev, staging, prod) y por qué
- **Infraestructura como código**: si aplica y con qué herramienta
- **Rollback**: cómo revertir un despliegue fallido de forma segura
- **Observabilidad en producción**: logging, métricas y alertas mínimas para operar el sistema

---

### 12. `$ARGUMENTS/analysis/060_tradeoffs.md`

Título: Decisiones y Trade-offs

Este documento es el más importante para el aprendizaje. Documenta las decisiones difíciles donde no hay una respuesta perfecta.

Incluye:
- **Las 5-7 decisiones más difíciles** de este diseño — las que tienen argumentos válidos en más de una dirección
- Para cada decisión:
  - El problema que forzó la decisión
  - Opción A: ventajas y desventajas
  - Opción B: ventajas y desventajas
  - La decisión tomada y el criterio desempate
  - Qué condición haría reconsiderar esta decisión en el futuro
- **Lecciones transferibles**: qué razonamientos de este proyecto aplican a otros sistemas similares

---

### 13. `$ARGUMENTS/analysis/065_tech_stack.md`

Título: Stack Tecnológico

Incluye:
- **Stack recomendado por capa**:
  - Frontend / Cliente
  - Backend / API
  - Base de datos principal
  - Base de datos secundaria (caché, búsqueda) si aplica
  - Infraestructura y cloud
  - Herramientas de observabilidad
- **Justificación de cada tecnología**: por qué esta y no su alternativa directa
- **Alternativas descartadas**: con razón específica para este proyecto
- **Compatibilidad del stack**: cómo encajan las piezas entre sí
- **Curva de aprendizaje y madurez**: consideraciones de equipo y ecosistema

---

### 14. `$ARGUMENTS/analysis/070_summary.md`

Título: Resumen Ejecutivo del Diseño

Incluye:
- **Visión general**: descripción concisa del sistema y sus objetivos de diseño
- **Las decisiones arquitectónicas clave**: las más importantes con su justificación en una línea cada una
- **Mapa de documentos**: una línea por cada documento generado indicando qué decisión principal contiene
- **Roadmap de implementación**: orden sugerido para construir el sistema por fases
- **Riesgos técnicos principales**: los 3-5 riesgos más importantes y cómo mitigarlos
- **Métricas de éxito**: cómo saber que el sistema funciona bien en producción

---

## Al terminar

Muestra al usuario la lista de archivos generados con una línea de descripción de cada uno. Indícale que puede hacer preguntas sobre cualquier concepto que no sea claro. Recuérdale que cuando esté listo, ejecute `/dsys-lessons $ARGUMENTS`
