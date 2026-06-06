# Estrategia de Despliegue — MeetingApp

## A diferencia de Contador_Calorias: aquí sí hay servidor que desplegar

*En Contador_Calorias, el "despliegue" era subir archivos estáticos a GitHub Pages — costo cero, sin servidores. MeetingApp tiene backend y base de datos, lo que añade complejidad real al despliegue.*

---

## Estrategia recomendada: Rolling Update

**¿Por qué Rolling Update y no Blue/Green?**

Blue/Green mantiene dos entornos idénticos activos simultáneamente — uno en producción, uno en espera — y redirige el tráfico instantáneamente. Para una app con 25 usuarios internos y downtime tolerable de minutos, el costo de mantener dos entornos duplicados (BD incluida) no se justifica.

Rolling Update actualiza instancias gradualmente. Con una sola instancia (que es lo adecuado para esta escala), el rolling update es simplemente: detener el proceso actual → desplegar el nuevo. Downtime: ~30 segundos máximo.

**¿Y Canary releases?** Canary tiene valor cuando hay millones de usuarios y se quiere exponer el cambio al 5% antes del 100%. Con 25 usuarios internos, una revisión de código + tests + deploy a staging es más efectivo que Canary.

---

## Containerización: Docker recomendado desde V1

Aunque la app podría correr sin contenedores, Docker resuelve el problema clásico de "funciona en mi máquina":

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
# docker-compose.yml (producción simple)
services:
  backend:
    build: ./backend
    env_file: .env
    restart: always
    depends_on:
      - db

  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data
    env_file: .env
    restart: always

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

**¿Kubernetes?** No para esta escala. Kubernetes gestiona clusters de múltiples nodos con alta disponibilidad. Para una app de empresa pequeña en un solo servidor, `docker compose up -d` es todo lo necesario.

---

## Pipeline CI/CD

```
Push a main
    │
    ├── 1. Lint (ruff/eslint)          ~30s
    ├── 2. Unit tests                  ~1min
    ├── 3. Integration tests (Docker)  ~3min
    ├── 4. Build Docker images         ~2min
    ├── 5. Deploy a staging            ~1min
    ├── 6. Smoke tests en staging      ~30s
    └── 7. Deploy a producción         ~1min
```

**Herramientas**: GitHub Actions (gratuito hasta ciertos límites).

**Regla crítica**: el paso 7 (producción) solo ocurre si todos los anteriores pasan. Las migraciones de BD se corren automáticamente al arrancar la app (con Alembic/Flyway), no como paso separado en el pipeline.

---

## Ambientes

| Ambiente | Propósito | Infraestructura |
|----------|----------|----------------|
| **Local** | Desarrollo activo | `docker compose up` con BD local |
| **Staging** | Verificar antes de producción | Mismo servidor que producción, diferente dominio |
| **Producción** | Uso real de la empresa | Servidor dedicado |

**¿Por qué staging si es una empresa de 25 personas?** El caso clásico de staging para este sistema: antes de desplegar el cambio que modifica la lógica de desplazamiento de prioridades, verificar en staging con datos reales que el comportamiento es el esperado. El costo de un bug en producción (una reunión con cliente externo sin sala) supera el costo de mantener staging.

---

## Rollback

**Con Docker**: revertir es trivial — desplegar la imagen anterior.

```bash
# Rollback: desplegar tag anterior
docker pull myregistry/meetingapp-backend:v1.2.0
docker compose up -d backend

# Rollback de BD: solo si la migración fue destructiva
# (Alembic/Flyway tienen comando downgrade)
alembic downgrade -1
```

**Regla**: nunca escribir migraciones destructivas (DROP COLUMN) sin un período de deprecación. Primero deprecar el columna (ignorarla en el código), luego eliminarla en la siguiente release.

---

## Observabilidad en producción

**Logging estructurado** (JSON logs — legibles por máquinas, no solo por humanos):

```json
{
  "timestamp": "2026-06-10T10:15:30Z",
  "level": "INFO",
  "event": "reservation_created",
  "reservation_id": "uuid",
  "sala_id": "uuid",
  "user_id": "uuid",
  "priority": "prioritaria",
  "duration_ms": 45
}
```

**Métricas mínimas** (con cualquier APM básico o simplemente logs):
- Latencia del endpoint `POST /reservas` (objetivo: < 200ms en P95)
- Tasa de errores 5xx
- Tiempo de respuesta de la BD (detecta queries lentas)

**Alertas mínimas**:
- Error rate > 1% durante 5 minutos → notificar a administradora
- BD no responde → restart automático del servicio

**Health-check endpoint** (también satisface el requerimiento OV-09 de notificación de restablecimiento):

```python
@app.route('/health')
def health():
    # Verifica que la BD responde
    db.execute("SELECT 1")
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}
```

Al arrancar, el sistema hace `GET /health` internamente. Si responde OK, dispara la notificación in-app a la administradora (ACP-09, ACP-12).
