# ACI606 - Docker Rescue Challenge

Aplicación web multi-servicio recuperada y ejecutable con un único comando de Docker Compose.

**Integrantes:** Guillermo Suitt, Benjamín Silva, Emilio Fernández · **Sección:** 401 · **Docente:** Oscar Loyola

## Arquitectura

```mermaid
flowchart LR
    U[Navegador] -->|"host :8080 → contenedor :80"| F["frontend<br/>Nginx 1.27"]
    F -->|"/api/* → http://backend:8000/"| B["backend<br/>FastAPI + uvicorn :8000"]
    B -->|"postgresql://…@db:5432/appdb"| D[("db<br/>PostgreSQL 16")]
    D --- V[("volumen pgdata")]
```

| Servicio | Imagen | Puerto interno | Publicado | Healthcheck |
|---|---|---|---|---|
| frontend | `./frontend` (nginx:1.27-alpine) | 80 | `${FRONTEND_PORT:-8080}` | — (espera a backend *healthy*) |
| backend | `./backend` (python:3.12-slim) | 8000 | no | `GET /health` (valida también la conexión a la BD) |
| db | `postgres:16-alpine` | 5432 | no | `pg_isready` por TCP |

- Los tres servicios comparten la red `docker-rescue-aci606_default` y se comunican por **nombre de servicio** (`backend`, `db`), nunca por `localhost`.
- Orden de arranque: `db` (healthy) → `backend` (healthy) → `frontend`.
- Los datos de PostgreSQL se guardan en el volumen nombrado `pgdata`, que sobrevive a `docker compose down`.

## Configuración

Los parámetros se leen desde un archivo `.env` (no versionado). La plantilla es `.env.example`:

| Variable | Valor por defecto | Uso |
|---|---|---|
| `POSTGRES_USER` | `appuser` | Usuario de PostgreSQL (db y backend) |
| `POSTGRES_PASSWORD` | `rescue123` | Contraseña de PostgreSQL (db y backend) |
| `POSTGRES_DB` | `appdb` | Base de datos |
| `FRONTEND_PORT` | `8080` | Puerto del host para el frontend |

Si no existe `.env`, `compose.yaml` usa los valores por defecto, por lo que el proyecto también arranca desde un Codespace limpio.

> PostgreSQL solo aplica `POSTGRES_USER` / `POSTGRES_PASSWORD` la primera vez que inicializa el volumen. Si se cambian después, hay que recrear el volumen con `docker compose down -v` (borra los datos).

## Uso

### 1. Iniciar

```bash
cp .env.example .env          # opcional
docker compose up --build -d
docker compose ps             # db y backend deben verse "(healthy)"
```

### 2. Verificar

```bash
curl http://localhost:8080/api/health
# {"status":"ok","database":"reachable"}

curl -X POST http://localhost:8080/api/items -H "Content-Type: application/json" -d '{"name":"prueba"}'
curl http://localhost:8080/api/items
```

En Codespaces, abrir la pestaña **Puertos** → puerto **8080** → icono del globo. La página debe mostrar `Backend: ok | DB: reachable`.

Logs de cada servicio:

```bash
docker compose logs frontend
docker compose logs backend
docker compose logs db
```

### 3. Detener

```bash
docker compose down           # elimina contenedores y red; CONSERVA el volumen pgdata
```

### 4. Recrear

```bash
docker compose up --build -d  # los datos guardados siguen disponibles
```

Para empezar desde cero (borra los datos):

```bash
docker compose down -v
docker compose up --build -d
```

## Pruebas de operación

| Prueba | Comandos | Resultado esperado |
|---|---|---|
| Persistencia | crear un item → `docker compose down` → `docker compose up -d` → `curl .../api/items` | El item sigue existiendo |
| Fallo controlado | `docker compose stop db` → `curl -i .../api/health` | `503`, la página muestra error; `docker compose logs backend` explica la causa |
| Recuperación | `docker compose start db` → esperar `healthy` → `curl .../api/health` | `200 OK` sin reiniciar el backend |

## Incidencias reparadas

| ID | Falla | Archivo |
|---|---|---|
| R-01 | `CMD` ejecutaba `uvicorn app:app`, pero el módulo es `main.py` | `backend/Dockerfile` |
| R-02 | Faltaba el driver `psycopg2-binary` | `backend/requirements.txt` |
| R-03 | `DATABASE_URL` apuntaba a `localhost` en vez de `db` | `compose.yaml` |
| R-04 | Contraseña `wrongpass` distinta a la de PostgreSQL | `compose.yaml` |
| R-05 | Puerto `8080:8080`, pero Nginx escucha en 80 | `compose.yaml` |
| R-06 | El backend arrancaba antes de que PostgreSQL estuviera listo | `compose.yaml` (healthcheck + `service_healthy`) |
| R-07 | `proxy_pass` de Nginx apuntaba a `localhost:8000` | `frontend/nginx.conf` |

Las evidencias (logs antes y después de cada reparación) están en `evidencias/`. El detalle de cada incidencia está en el informe técnico.
