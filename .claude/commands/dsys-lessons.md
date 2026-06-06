Extrae las lecciones aprendidas del proyecto y las guarda en la base de datos local, sin repetir lo que ya está registrado.

El nombre del proyecto es: $ARGUMENTS

## Instrucciones

### Paso 1 — Inicializar la base de datos

Ejecuta este comando para crear `lessons.db` si no existe (tabla principal + índice FTS5 + triggers de sincronización):

```bash
python -c "
import sqlite3
conn = sqlite3.connect('lessons.db')

conn.execute('''
    CREATE TABLE IF NOT EXISTS lessons (
        id          INTEGER PRIMARY KEY AUTOINCREMENT,
        project     TEXT NOT NULL,
        title       TEXT NOT NULL,
        context     TEXT NOT NULL,
        learned     TEXT NOT NULL,
        why_matters TEXT NOT NULL,
        example     TEXT NOT NULL,
        tags        TEXT DEFAULT '',
        created_at  TEXT NOT NULL DEFAULT (datetime('now', 'localtime'))
    )
''')

conn.execute('''
    CREATE VIRTUAL TABLE IF NOT EXISTS lessons_fts USING fts5(
        title,
        context,
        learned,
        why_matters,
        example,
        tags,
        content=lessons,
        content_rowid=id
    )
''')

conn.execute('''
    CREATE TRIGGER IF NOT EXISTS lessons_ai
    AFTER INSERT ON lessons BEGIN
        INSERT INTO lessons_fts(rowid, title, context, learned, why_matters, example, tags)
        VALUES (new.id, new.title, new.context, new.learned, new.why_matters, new.example, new.tags);
    END
''')

conn.execute('''
    CREATE TRIGGER IF NOT EXISTS lessons_ad
    AFTER DELETE ON lessons BEGIN
        INSERT INTO lessons_fts(lessons_fts, rowid, title, context, learned, why_matters, example, tags)
        VALUES ('delete', old.id, old.title, old.context, old.learned, old.why_matters, old.example, old.tags);
    END
''')

conn.execute('''
    CREATE TRIGGER IF NOT EXISTS lessons_au
    AFTER UPDATE ON lessons BEGIN
        INSERT INTO lessons_fts(lessons_fts, rowid, title, context, learned, why_matters, example, tags)
        VALUES ('delete', old.id, old.title, old.context, old.learned, old.why_matters, old.example, old.tags);
        INSERT INTO lessons_fts(rowid, title, context, learned, why_matters, example, tags)
        VALUES (new.id, new.title, new.context, new.learned, new.why_matters, new.example, new.tags);
    END
''')

conn.commit()
conn.close()
print('OK')
"
```

---

### Paso 2 — Obtener títulos ya registrados

Ejecuta este comando para saber qué lecciones ya existen y evitar duplicados:

```bash
python -c "
import sqlite3
conn = sqlite3.connect('lessons.db')
rows = conn.execute('SELECT title FROM lessons').fetchall()
conn.close()
for r in rows:
    print(r[0])
"
```

Guarda mentalmente esa lista — son los conceptos que NO debes repetir.

---

### Paso 3 — Leer los documentos de análisis

Lee todos los archivos en `$ARGUMENTS/analysis/`:
- 005_architecture.md
- 010_database_model.md
- 015_design_patterns.md
- 020_data_structures.md
- 025_communication_protocols.md
- 030_scalability.md
- 035_advanced_concepts.md
- 040_maintainability.md
- 045_security.md
- 050_testing_strategy.md
- 055_deployment.md
- 060_tradeoffs.md
- 065_tech_stack.md
- 070_summary.md

---

### Paso 4 — Identificar lecciones nuevas

Compara los conceptos clave del análisis contra los títulos ya registrados en la BD.

Extrae solo los conceptos **genuinamente nuevos** — que no hayan aparecido antes, ni siquiera de forma similar.

Para cada lección nueva define:
- **title**: nombre corto del concepto (ej: "SELECT FOR UPDATE para concurrencia")
- **context**: en qué tipo de sistema o situación aplica
- **learned**: la idea central explicada con claridad
- **why_matters**: consecuencia práctica de aplicarlo o ignorarlo
- **example**: cómo apareció específicamente en $ARGUMENTS
- **tags**: 3-5 palabras clave separadas por coma (ej: base-de-datos,concurrencia,postgresql)

---

### Paso 5 — Mostrar en pantalla

Antes de guardar, muestra las lecciones nuevas con este formato:

```
## Lecciones aprendidas — $ARGUMENTS

Encontré X concepto(s) nuevo(s):

**[título]**
Contexto: ...
Qué aprendí: ...
Por qué importa: ...
Ejemplo del proyecto: ...
Tags: ...
```

Si no hay conceptos nuevos, indica:
```
No encontré conceptos nuevos. Todo ya está registrado en lessons.db.
```
Y detente aquí — no ejecutes el Paso 6.

---

### Paso 6 — Guardar en la base de datos

Por cada lección nueva, ejecuta un comando de inserción. Reemplaza los valores reales donde dice VALOR:

```bash
python -c "
import sqlite3
conn = sqlite3.connect('lessons.db')
conn.execute(
    'INSERT INTO lessons (project, title, context, learned, why_matters, example, tags) VALUES (?,?,?,?,?,?,?)',
    ('PROYECTO', 'TITLE', 'CONTEXT', 'LEARNED', 'WHY_MATTERS', 'EXAMPLE', 'TAGS')
)
conn.commit()
conn.close()
print('Guardado: TITLE')
"
```

Ejecuta un comando separado por cada lección nueva.

---

### Paso 7 — Confirmar

Muestra el total de lecciones guardadas en esta sesión y el total acumulado en la BD:

```bash
python -c "
import sqlite3
conn = sqlite3.connect('lessons.db')
total = conn.execute('SELECT COUNT(*) FROM lessons').fetchone()[0]
proyecto = conn.execute('SELECT COUNT(*) FROM lessons WHERE project = ?', ('$ARGUMENTS',)).fetchone()[0]
conn.close()
print(f'Lecciones de $ARGUMENTS: {proyecto}')
print(f'Total acumulado en BD: {total}')
"
```

---

### Paso 8 — Subir a GitHub

```bash
git add -A && git commit -m "lessons($ARGUMENTS): agrega lecciones aprendidas del proyecto" && git push
```

Si el push falla por falta de cambios (`nothing to commit`), ignóralo — significa que no hubo lecciones nuevas. Confirma al usuario que todo está sincronizado con GitHub.
