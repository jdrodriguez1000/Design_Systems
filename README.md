# Design Systems — Aprendizaje de Diseño de Sistemas

Herramienta educativa para aprender diseño de sistemas a través de proyectos reales.
Por cada proyecto se generan documentos que explican las decisiones arquitectónicas,
sus justificaciones, trade-offs y conceptos aplicados.

---

## Estructura

```
Design_Systems/
├── lessons.db                  ← base de datos SQLite con todas las lecciones
├── .claude/
│   └── commands/
│       ├── dsys-scaffold.md    ← comando para crear un proyecto nuevo
│       ├── dsys-analyze.md     ← comando para analizar los requerimientos
│       ├── dsys-lessons.md     ← comando para extraer y guardar lecciones
│       └── dsys-search.md      ← comando para buscar lecciones aprendidas
└── NombreProyecto/
    ├── requirements/
    │   └── requirements.md     ← requerimientos funcionales y no funcionales
    └── analysis/
        ├── 005_architecture.md
        ├── 010_database_model.md
        ├── 015_design_patterns.md
        ├── 020_data_structures.md
        ├── 025_communication_protocols.md
        ├── 030_scalability.md
        ├── 035_advanced_concepts.md
        ├── 040_maintainability.md
        ├── 045_security.md
        ├── 050_testing_strategy.md
        ├── 055_deployment.md
        ├── 060_tradeoffs.md
        ├── 065_tech_stack.md
        └── 070_summary.md
```

---

## Flujo de trabajo

### 1. Crear un proyecto nuevo

```
/dsys-scaffold "Nombre del Proyecto"
```

Crea la estructura de carpetas y un template de requerimientos listo para llenar.

### 2. Definir los requerimientos

Edita el archivo generado:

```
NombreProyecto/requirements/requirements.md
```

Completa las secciones: descripción general, requerimientos funcionales,
requerimientos no funcionales, escala esperada y restricciones.

### 3. Generar el análisis

```
/dsys-analyze "Nombre del Proyecto"
```

El agente lee los requerimientos y genera 8 documentos educativos en la carpeta `analysis/`.

### 4. Estudiar y preguntar

Revisa los documentos generados. Usa Claude Code para hacer preguntas,
pedir aclaraciones o profundizar en cualquier concepto que no sea claro.

### 5. Registrar lecciones aprendidas

```
/dsys-lessons "Nombre del Proyecto"
```

El agente extrae los conceptos nuevos aprendidos en este proyecto,
los muestra en pantalla y los guarda en `lessons.db` sin repetir
lo que ya estaba registrado de proyectos anteriores.

### 6. Buscar lecciones

```
/dsys-search "término"
```

Busca en `lessons.db` por palabra clave, tag o nombre de proyecto.
Ejemplos: `/dsys-search cache`, `/dsys-search seguridad`, `/dsys-search Test001`

---

## Documentos de análisis

| Archivo | Contenido |
|---|---|
| `005_architecture.md` | Estilo arquitectónico recomendado, justificación y alternativas |
| `010_database_model.md` | Tipo de base de datos, esquema, indexación, ACID vs BASE |
| `015_design_patterns.md` | Patrones creacionales, estructurales, de comportamiento y arquitectónicos |
| `020_data_structures.md` | Estructuras clave para el dominio con complejidad y casos de uso |
| `025_communication_protocols.md` | REST vs gRPC vs GraphQL, mensajería, tiempo real |
| `030_scalability.md` | Caché, sharding, load balancing, cuellos de botella |
| `035_advanced_concepts.md` | Teorema CAP, consistencia, ACID vs BASE |
| `040_maintainability.md` | SOLID, modularidad, deuda técnica, versionado |
| `045_security.md` | Amenazas, autenticación, cifrado, OWASP, auditoría |
| `050_testing_strategy.md` | Pirámide de testing, qué testear, contract testing |
| `055_deployment.md` | CI/CD, containerización, ambientes, rollback |
| `060_tradeoffs.md` | Las decisiones difíciles con argumentos en ambas direcciones |
| `065_tech_stack.md` | Stack por capa con justificación y alternativas descartadas |
| `070_summary.md` | Resumen ejecutivo, roadmap y riesgos |

---

## Requisitos

- Claude Code CLI instalado (`claude`)
- Directorio abierto en terminal: `Design_Systems/`
