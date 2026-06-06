# Estrategia de Despliegue — InventariApp

## A diferencia de Contador_Calorias: aquí sí hay servidor y base de datos que operar

*En Contador_Calorias, el "despliegue" era subir archivos estáticos a GitHub Pages — cero servidores. InventariApp tiene backend y base de datos, lo que añade complejidad real al despliegue. Sin embargo, es más simple que MeetingApp porque no hay lógica de alta disponibilidad crítica — la empresa ya tiene un fallback manual documentado.*

---

## Estrategia recomendada: Rolling Update simple

Con una sola instancia del backend (adecuado para esta escala), el rolling update es: detener el proceso actual → ejecutar migraciones pendientes → arrancar el nuevo proceso. Downtime: ~30 segundos máximo.

**¿Por qué no Blue/Green?**
Blue/Green mantiene dos entornos idénticos simultáneos y redirige el tráfico instantáneamente. Para una empresa de 18 personas con un fallback manual documentado (papel + WhatsApp con Ana), el costo de mantener dos entornos duplicados incluyendo BD no se justifica.

**¿Por qué no Canary?**
Canary tiene valor cuando hay miles de usuarios y se quiere exponer el cambio al 5% antes del 100%. Con 18 usuarios internos, una revisión de código + tests + deploy a staging es más efectivo.

---

## Containerización: Docker Compose desde v1

Docker resuelve el problema clásico de "funciona en mi máquina" y facilita tener staging idéntico a producción:

```dockerfile
# Backend
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml (producción)
services:
  backend:
    build: ./backend
    env_file: .env
    restart: always
    depends_on:
      db:
        condition: service_healthy
    command: >
      sh -c "alembic upgrade head && uvicorn main:app --host 0.0.0.0 --port 8000"
    # Las migraciones corren al arrancar — simples y predecibles

  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data
    env_file: .env
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER"]
      interval: 5s
      timeout: 5s
      retries: 5

  frontend:
    build: ./frontend
    restart: always

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/letsencrypt
    restart: always

volumes:
  postgres_data:
```

**¿Kubernetes?** No para esta escala. Docker Compose es todo lo que se necesita. Kubernetes gestiona clusters de múltiples nodos con alta disponibilidad — para una app de empresa pequeña en un solo servidor, es sobrediseño operacional.

---

## Pipeline CI/CD

```
Push a main
    │
    ├── 1. Lint (ruff/eslint)            ~30s
    ├── 2. Unit tests                    ~1min
    ├── 3. Integration tests (Docker)    ~3min
    ├── 4. Build Docker images           ~2min
    ├── 5. Deploy a staging              ~1min
    ├── 6. Smoke tests en staging        ~30s
    │      (POST /movimientos, GET /alertas, GET /dashboard/kpis)
    └── 7. Deploy a producción           ~1min
```

**Herramientas**: GitHub Actions (gratuito hasta ciertos límites).

**Regla crítica**: el paso 7 (producción) solo ocurre si todos los anteriores pasan. Las migraciones de BD corren al arrancar el proceso (via `alembic upgrade head` en el CMD del Docker) — no como paso separado del pipeline.

**Smoke tests mínimos en staging**:
1. `POST /movimientos` con un movimiento de prueba → verifica que el backend responde y la BD está accesible
2. `GET /dashboard/kpis` → verifica que el cálculo de KPIs ejecutivos responde
3. `GET /alertas?estado=activa` → verifica la query de alertas

---

## Ambientes

| Ambiente | Propósito | Infraestructura |
|----------|----------|----------------|
| **Local** | Desarrollo activo | `docker compose up` con BD local |
| **Staging** | Verificar antes de producción | Mismo servidor que producción, diferente subdominio |
| **Producción** | Uso real de la empresa | Servidor dedicado |

**¿Por qué staging si es una empresa de 18 personas?** El caso concreto para este sistema: antes de desplegar el cambio que modifica la lógica de evaluación de alertas (la parte más crítica), verificar en staging con datos de prueba que el comportamiento es el esperado. El costo de un bug en producción (alertas que no se generan → stock agotado sin aviso) supera el costo de mantener staging.

---

## Rollback

**Con Docker**: revertir es trivial — desplegar la imagen anterior.

```bash
# Rollback: desplegar tag anterior
docker pull myregistry/inventariapp-backend:v1.2.0
docker compose up -d backend

# Rollback de BD (solo si la migración fue destructiva)
alembic downgrade -1
```

**Regla**: nunca escribir migraciones destructivas (DROP COLUMN) sin un período de deprecación. Primero deprecar el campo (ignorarlo en el código), luego eliminarlo en la siguiente release.

**Caso especial para el ledger**: la tabla `movimientos` es append-only por diseño. Una migración que altere esta tabla debe tener rollback explícito planificado antes de correr en producción.

---

## Observabilidad en producción

**Logging estructurado JSON** (legible por máquinas y herramientas de análisis):

```json
{
  "timestamp": "2026-06-06T14:30:00Z",
  "level": "INFO",
  "event": "movement_registered",
  "movement_id": "uuid",
  "producto_id": "uuid",
  "tipo_impacto": "RESTA",
  "cantidad": 30,
  "nuevo_stock": 5,
  "alerta_generada": true,
  "duration_ms": 23
}
```

**Métricas mínimas**:
- Latencia del endpoint `POST /movimientos` (objetivo: < 200ms en P95)
- Tasa de errores 5xx
- Número de alertas activas actuales (métrica de negocio)

**Alertas técnicas mínimas**:
- Error rate > 1% durante 5 minutos → notificar a Andrés o administrador del sistema
- BD no responde → restart automático del servicio

**Health-check endpoint**:

```python
@app.get('/health')
def health():
    db.execute("SELECT 1")  # Verifica que la BD responde
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}
```

---

## Backups — críticos para el ledger inmutable

El historial de movimientos es la fuente de verdad del inventario. Sin backups, una corrupción de la BD es irrecuperable.

```bash
# Backup diario automatizado (crontab o GitHub Actions scheduled)
pg_dump $DATABASE_URL | gzip > backup_$(date +%Y%m%d).sql.gz
# Subir a S3 o Google Cloud Storage

# Retención sugerida: 30 días de backups diarios
```

Esta es posiblemente la acción de infraestructura más importante del proyecto. Un sistema de inventario sin backups convierte cada falla de disco en una pérdida total del historial de operaciones del negocio.
