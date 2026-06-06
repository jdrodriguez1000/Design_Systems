Busca lecciones aprendidas en la base de datos local usando el índice FTS5.

El término de búsqueda es: $ARGUMENTS

## Instrucciones

### Paso 1 — Verificar que la BD existe

```bash
python -c "
import sqlite3, sys
from pathlib import Path
if not Path('lessons.db').exists():
    print('ERROR: lessons.db no encontrada. Ejecuta primero /dsys-lessons en algun proyecto.')
    sys.exit(1)
print('OK')
"
```

Si retorna ERROR, detente e informa al usuario.

---

### Paso 2 — Ejecutar la búsqueda con FTS5

```bash
python -c "
import sqlite3, sys

term = '$ARGUMENTS'
conn = sqlite3.connect('lessons.db')

# FTS5 MATCH con prefijo para encontrar variantes de la palabra
# Ejemplo: 'cache*' encuentra cache, caches, cacheable, etc.
# Si el término tiene espacios se convierte en búsqueda de frase
words = term.strip().split()
if len(words) == 1:
    query = words[0] + '*'
else:
    query = ' '.join(w + '*' for w in words)

try:
    rows = conn.execute('''
        SELECT l.project, l.title, l.context, l.learned,
               l.why_matters, l.example, l.tags, l.created_at
        FROM lessons l
        JOIN lessons_fts ON l.id = lessons_fts.rowid
        WHERE lessons_fts MATCH ?
        ORDER BY rank
    ''', (query,)).fetchall()
except sqlite3.OperationalError as e:
    print(f'Error en busqueda: {e}')
    sys.exit(1)

conn.close()

if not rows:
    print(f'Sin resultados para: {term}')
else:
    print(f'{len(rows)} resultado(s) para \"{term}\":\n')
    for r in rows:
        print(f'Proyecto : {r[0]}  |  Tags: {r[6]}  |  Fecha: {r[7][:10]}')
        print(f'Titulo   : {r[1]}')
        print(f'Contexto : {r[2]}')
        print(f'Aprendi  : {r[3]}')
        print(f'Importa  : {r[4]}')
        print(f'Ejemplo  : {r[5]}')
        print('-' * 60)
"
```

---

### Paso 3 — Presentar resultados

Muestra los resultados al usuario con formato claro. Si hay más de 5 resultados, agrúpalos por proyecto.

Si no hay resultados, sugiere al usuario:
- Usar un término más corto o general
- Buscar por nombre de proyecto (ej: `Test001`)
- Buscar por tag: `arquitectura`, `base-de-datos`, `seguridad`, `escalabilidad`, `patrones`, `testing`, `deployment`, `concurrencia`
