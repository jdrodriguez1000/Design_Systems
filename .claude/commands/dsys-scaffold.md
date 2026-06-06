Crea la estructura de carpetas y archivos para un nuevo proyecto de diseño de sistemas.

El nombre del proyecto es: $ARGUMENTS

## Instrucciones

1. Crea la carpeta `$ARGUMENTS/requirements/`
2. Crea la carpeta `$ARGUMENTS/analysis/`
3. Crea los siguientes 8 archivos en `$ARGUMENTS/requirements/` con exactamente el contenido indicado para cada uno:

---

### `$ARGUMENTS/requirements/shared_understanding.md`

```
# Shared Understanding — $ARGUMENTS

## Propósito del Sistema
<!-- Qué problema resuelve y para quién -->

## Actores y sus Necesidades
<!-- Quién usa el sistema y qué necesita cada uno -->

## Flujos Principales
<!-- Cómo usa el sistema cada actor — paso a paso -->

## Lo que el Sistema NO Hace
<!-- Exclusiones explícitas del scope -->
```

---

### `$ARGUMENTS/requirements/domain_glossary.md`

```
# Domain Glossary — $ARGUMENTS

| Término | Definición acordada | Sinónimos a evitar | Actor que lo usa |
|---------|--------------------|--------------------|-----------------|
|         |                    |                    |                 |
```

---

### `$ARGUMENTS/requirements/scope_boundaries.md`

```
# Scope Boundaries — $ARGUMENTS

## Qué NO hará el sistema

| ID | Exclusión | Razón |
|----|-----------|-------|
|    |           |       |

## Qué queda diferido (posibles fases futuras)

| ID | Capacidad diferida | Condición para incluir |
|----|--------------------|----------------------|
|    |                    |                      |
```

---

### `$ARGUMENTS/requirements/failure_behavior.md`

```
# Failure Behavior — $ARGUMENTS

## Escenarios de Fallo por Actor

### [Nombre Actor]

| ID | Escenario | Causa probable | Comportamiento esperado | Prioridad |
|----|-----------|---------------|------------------------|-----------|
|    |           |               |                        |           |
```

---

### `$ARGUMENTS/requirements/acceptance_criteria.md`

```
# Acceptance Criteria — $ARGUMENTS

## Criterios de Aceptación por Actor

### [Nombre Actor]

| ID | Criterio | Condición de cumplimiento | Condición de fallo |
|----|----------|--------------------------|-------------------|
|    |          |                          |                   |
```

---

### `$ARGUMENTS/requirements/bdd_features.md`

```
# BDD Features — $ARGUMENTS

## [Nombre Feature]

**Como** [actor]
**Quiero** [acción]
**Para** [beneficio]

### Escenario: [nombre escenario]
**Dado** [contexto inicial]
**Cuando** [acción del usuario]
**Entonces** [resultado esperado]
```

---

### `$ARGUMENTS/requirements/data_contracts.md`

```
# Data Contracts — $ARGUMENTS

## Entidades

### [Nombre Entidad]
**Descripción:** 

| Campo | Tipo de dato | Obligatorio | Validaciones de negocio |
|-------|-------------|-------------|------------------------|
|       |             |             |                        |
```

---

### `$ARGUMENTS/requirements/error_exception_policy.md`

```
# Error & Exception Policy — $ARGUMENTS

## Políticas por Actor

### [Nombre Actor]

| ID | Escenario de error | Causa | Mensaje al usuario | Reintento | Bloqueo | Acción alternativa |
|----|-------------------|-------|-------------------|-----------|---------|-------------------|
|    |                   |       |                   |           |         |                   |
```

---

4. Confirma al usuario que la estructura fue creada y muestra el árbol de archivos generados.
5. Indícale que el siguiente paso es llenar los 8 archivos en `$ARGUMENTS/requirements/` con los requerimientos del proyecto.
6. Recuérdale que cuando termine, ejecute `/dsys-analyze $ARGUMENTS`
