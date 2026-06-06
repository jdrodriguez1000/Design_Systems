# CLAUDE.md — Design Systems

Este directorio es una herramienta educativa de diseño de sistemas.
El usuario está aprendiendo arquitectura de software a través de proyectos reales.

## Propósito

Por cada proyecto el usuario define requerimientos y se generan documentos que explican
las decisiones arquitectónicas con sus justificaciones, trade-offs y conceptos aplicados.
El objetivo es educativo: el usuario debe entender el POR QUÉ de cada decisión, no solo el QUÉ.

## Comandos disponibles

| Comando | Función |
|---|---|
| `/dsys-scaffold "Proyecto"` | Crea estructura de carpetas y los 8 templates de requerimientos |
| `/dsys-analyze "Proyecto"` | Lee los 8 archivos de requerimientos y genera 14 documentos de análisis |
| `/dsys-lessons "Proyecto"` | Extrae lecciones nuevas y las guarda en `lessons.db` |
| `/dsys-search "término"` | Busca lecciones en `lessons.db` por palabra clave, tag o proyecto |

## Estructura de un proyecto

```
NombreProyecto/
├── requirements/    ← 8 archivos de requerimientos
└── analysis/        ← 14 documentos educativos generados
```

## Base de datos de aprendizaje

`lessons.db` en la raíz acumula las lecciones de todos los proyectos.
Usa FTS5 para búsqueda eficiente. `dsys-analyze` la consulta para informar el análisis de cada proyecto nuevo.

## Comportamiento esperado al responder preguntas

- Explica siempre el POR QUÉ, no solo el QUÉ
- Menciona las alternativas y sus trade-offs
- Usa ejemplos de sistemas reales (Netflix, Uber, Amazon, etc.) cuando sea útil
- Adapta la profundidad a las dudas del usuario
- Conecta los conceptos con los requerimientos específicos del proyecto

---

## PRINCIPIOS DE COMPORTAMIENTO

**PI-1. Razona antes de actuar.** Debes exponer pros, contras y suposiciones. Ante ambigüedad, detente y consulta; nunca elijas en silencio.

**PI-2. Simplicidad primero.** Código mínimo con interfaces simples. Sin abstracciones, parámetros ni configurabilidad no solicitados.

**PI-3. Cambios quirúrgicos.** Solo toca lo necesario para la tarea. No refactorices lo que funciona. No borres código muerto preexistente sin autorización.

**PI-4. Slices verticales.** Una funcionalidad completa (datos→interfaz) a la vez. Valida integración con un "Tracer Bullet" antes de ampliar.

**PI-5. Orientado a comportamiento.** Toda tarea tiene un test que la respalda. Definición de Terminado = test en verde. Sin excepción.

## Notas de comportamiento para el agente que retome

- No reabrir decisiones cerradas de la tabla de arriba
- El usuario confirma antes de que se escriba al archivo — proponer primero si hay algo que decidir
- Respuestas cortas y directas — el usuario no necesita explicaciones largas de lo que ya conoce
- Cuando el usuario dice "sigamos", "adelante", "si" o "A/B/C", escribir directamente sin pedir más confirmación
- El doc_evaluator fue descrito por el usuario como "prueba de stress documental" — esa es la mentalidad correcta, pero adaptada a este proyecto
