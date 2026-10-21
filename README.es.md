# Whaler — Docker Compose

[English](README.md) | **Español**

Este repositorio arranca todo el stack de **Whaler** con Docker Compose. Whaler es una aplicación web para subir vídeos de cetáceos y procesarlos con IA: detecta las ballenas, las recorta, detecta sus marcas y clasifica la especie. El frontend, el backend y el worker están en sus propios repositorios. Este solo contiene el `compose.yaml` que los construye y los conecta con los servicios de datos (PostgreSQL, Redis y MinIO).

## Repositorios

| Repositorio | Carpeta | Qué es |
|---|---|---|
| [aitoralco/tfg_frontend](https://github.com/aitoralco/tfg_frontend) | `tfg_frontend` | Cliente web (Angular + Nginx) |
| [aitoralco/tfg_backend](https://github.com/aitoralco/tfg_backend) | `tfg_backend` | API REST (FastAPI), migraciones y seed de la BD |
| [aitoralco/tfg_worker](https://github.com/aitoralco/tfg_worker) | `tfg_worker` | Worker de procesamiento con IA (RQ + YOLO) |
| [aitoralco/tfg_compose](https://github.com/aitoralco/tfg_compose) | `tfg_dockercompose` (cualquier nombre) | Este repositorio |

`compose.yaml` construye las imágenes desde `../tfg_frontend`, `../tfg_backend` y `../tfg_worker`. **Los cuatro repositorios tienen que estar clonados uno al lado del otro en la misma carpeta, con esos nombres exactos**:

```
code/
├── tfg_backend/
├── tfg_frontend/
├── tfg_worker/
└── tfg_dockercompose/   # este repositorio
```

## Requisitos

- **Docker** con **Compose v2** (el comando `docker compose`): Docker Desktop en Windows/macOS, o Docker Engine + el plugin de Compose en Linux.
- **Git**, para clonar los cuatro repositorios.
- **Los modelos YOLO entrenados para el worker** (ver más abajo). Sin ellos el stack arranca, pero no se puede procesar ningún vídeo.
- **Puertos libres** en el host: `8080` (frontend), `8000` (backend), `5432` (PostgreSQL), `6379` (Redis), `9000` y `9001` (MinIO).
- **Espacio en disco y memoria.** La imagen del worker es la más grande: incluye PyTorch (versión CPU), OpenCV y ffmpeg. Los vídeos y sus resultados se guardan en volúmenes de Docker.
- No hace falta GPU: el worker funciona en CPU (el procesamiento es lento).

### Modelos del worker

> **Los pesos de los modelos entrenados no están en ningún repositorio.** Se entrenaron con imágenes que no pertenecen al autor: se cedieron únicamente para este Trabajo de Fin de Grado (TFG), así que ni las imágenes ni los modelos entrenados con ellas se pueden distribuir.

Para procesar vídeos, coloca tus propios pesos en el repositorio del worker antes de construir:

```
tfg_worker/app/ai/trained_models/
├── detect_whales/best.pt     # detección de cetáceos (OBB)   → DW_MODEL_PATH
├── detect_marks/best.pt      # detección de marcas           → DEM_MODEL_PATH
└── classify_whales/best.pt   # clasificación de especies     → CW_MODEL_PATH
```

La imagen del worker copia esta carpeta (está ignorada por git, pero no por Docker). Las variables `*_MODEL_PATH` del `.env` apuntan a estos ficheros dentro del contenedor (`/worker/app/ai/...`). Si falta un modelo, todos los vídeos que se suban acaban con estado `error` en la fase DW. Más detalles en el [README del worker](https://github.com/aitoralco/tfg_worker/blob/main/README.es.md#modelos).

## Arquitectura

| Servicio | Imagen / build | Puerto en el host | Función |
|---|---|---|---|
| `frontend-angular` | `../tfg_frontend` | `8080` | Sirve la app Angular con Nginx |
| `backend-fastapi` | `../tfg_backend` | `8000` | API REST: usuarios, subida y streaming de vídeos, resultados de la IA |
| `backend-migrate` | `../tfg_backend` | — | Tarea de un solo uso: crea el esquema de la BD y los datos base, y termina |
| `worker-rq` | `../tfg_worker` | — | Recoge los trabajos de Redis y procesa los vídeos con los modelos YOLO |
| `db-postgres` | `postgres:15-alpine` | `5432` | BD relacional: usuarios, roles, vídeos y su estado |
| `cache-redis` | `redis:latest` | `6379` | Cola de trabajos (RQ) entre el backend y el worker |
| `filesystem-minio` | `quay.io/minio/minio:latest` | `9000` / `9001` | Almacenamiento de objetos compatible con S3: vídeos, miniaturas y resultados de la IA (API / consola web) |

```
                       navegador
       ┌───────────────────┼─────────────────────────┐
 :8080 │ app         :8000 │ API, streaming     :9000 │ URLs firmadas
       ▼                   ▼                         ▼
 frontend-angular    backend-fastapi ──────────▶ filesystem-minio
                       │         │  ficheros         ▲
                   SQL │         │ encola trabajo    │ baja vídeo, sube resultados
                       ▼         ▼                   │
                 db-postgres   cache-redis ──job─▶ worker-rq
                       ▲                             │
                       └───── estado del vídeo ──────┘
```

Flujo de procesamiento:

1. El usuario sube un vídeo desde el frontend. El backend lo guarda en MinIO, crea la fila en PostgreSQL y encola `process_video(video_id)` en Redis.
2. El worker recoge el trabajo, descarga el vídeo de MinIO y ejecuta las fases DW → EW → DEM → CW. Actualiza el estado del vídeo en la BD en cada fase y sube los resultados a MinIO.
3. El frontend pide al backend el estado y los resultados (vídeo anotado, recortes, clasificación).

### Por qué arranca así

Los servicios arrancan en este orden, controlado con `depends_on` y healthchecks:

```
db-postgres (healthy) ──▶ backend-migrate (termina con código 0) ──┬──▶ backend-fastapi ──▶ frontend-angular
cache-redis (healthy) ─────────────────────────────────────────────┤
filesystem-minio (healthy) ────────────────────────────────────────┴──▶ worker-rq
```

- **Healthchecks, no solo "arrancado".** Un contenedor puede estar en marcha antes de que su servicio acepte conexiones. PostgreSQL (`pg_isready`), Redis (`redis-cli ping`) y MinIO (`/minio/health/live`) tienen healthchecks, y los servicios que los usan esperan a `service_healthy`.
- **Una tarea de migración aparte.** `backend-migrate` usa la imagen del backend, pero ejecuta `alembic upgrade head` (crea o actualiza el esquema) y después `python -m app.db.starter` (seed: roles, estados de vídeo y el usuario admin de `ADMIN_*`), y termina. El backend y el worker esperan a que acabe bien (`service_completed_successfully`). Así:
  - las tablas existen antes de que nadie las consulte,
  - las migraciones se ejecutan una sola vez, no en cada réplica del backend,
  - el worker encuentra los estados de vídeo, que busca por nombre (`processing_dw`, `processed`…),
  - si una migración falla, el backend y el worker no arrancan, y el error aparece en `docker compose logs backend-migrate`.
- **Configuración compartida.** El backend y la migración usan las mismas variables de entorno (el bloque `x-backend-env`), así que siempre se conectan a la misma BD.
- **El bucket lo crea el backend.** Si `BUCKET_NAME` no existe en MinIO, el backend lo crea la primera vez que se conecta.
- **Dos URLs de MinIO.** Dentro de Docker, el backend y el worker llegan a MinIO como `http://filesystem-minio:9000`, pero ese nombre no existe para el navegador. Por eso el backend firma las URLs de los ficheros con `FILESYSTEM_PUBLIC_URL` (`http://localhost:9000`).
- **El frontend llama al backend desde el navegador**, no desde dentro de Docker. Su URL de la API (`http://127.0.0.1:8000`) queda fijada al construir la imagen. Para servirlo desde otro host, cambia `tfg_frontend/src/environments/environment.ts`, añade el origen a la lista de CORS del backend y vuelve a construir.

### Persistencia

Los datos se guardan en volúmenes de Docker con nombre, así que sobreviven a `docker compose down`:

| Volumen | Servicio | Contenido |
|---|---|---|
| `postgres_data` | `db-postgres` | Base de datos |
| `redis_data` | `cache-redis` | Datos de Redis (cola) |
| `minio_data` | `filesystem-minio` | Vídeos, miniaturas y resultados de la IA |

Redis carga su configuración de `redis/redis.conf`. La contraseña no está en ese fichero: se pasa con `--requirepass ${REDIS_PASSWORD}`.

## Puesta en marcha

1. **Clona los repositorios** uno al lado del otro:
   ```bash
   mkdir whaler && cd whaler
   git clone https://github.com/aitoralco/tfg_backend.git
   git clone https://github.com/aitoralco/tfg_frontend.git
   git clone https://github.com/aitoralco/tfg_worker.git
   git clone https://github.com/aitoralco/tfg_compose.git tfg_dockercompose
   cd tfg_dockercompose
   ```
2. **Añade los modelos** en `../tfg_worker/app/ai/trained_models/` (ver [Modelos del worker](#modelos-del-worker)).
3. **Crea el `.env`** a partir del ejemplo y rellena los valores:
   ```bash
   cp .env.example .env
   ```

   | Variable | Uso |
   |---|---|
   | `DB_NAME`, `DB_USER`, `DB_PASSWORD` | Base de datos y credenciales de PostgreSQL |
   | `ADMIN_USERNAME`, `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Usuario admin que crea el seed (la contraseña se guarda hasheada) |
   | `REDIS_PASSWORD` | Contraseña de Redis |
   | `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD` | Credenciales de MinIO (la contraseña necesita al menos 8 caracteres) |
   | `BUCKET_NAME` | Bucket donde se guarda todo (p. ej. `videos-bucket`) |
   | `JWT_SECRET_KEY` | Clave para firmar los tokens de login (una cadena larga y aleatoria) |
   | `DW_MODEL_PATH`, `DEM_MODEL_PATH`, `CW_MODEL_PATH` | Rutas de los modelos dentro del contenedor del worker (los valores por defecto sirven) |

4. **Construye y arranca** todo:
   ```bash
   docker compose up --build -d
   ```
   La primera construcción tarda (sobre todo el worker, por PyTorch).
5. **Comprueba el estado:**
   ```bash
   docker compose ps
   ```
   Todos los servicios deberían estar `running`, y los que tienen healthcheck, `healthy`. `backend-migrate` aparece como `exited (0)`: es lo esperado, solo se ejecuta una vez al arrancar.
6. **Abre la app:**

   | URL | Qué |
   |---|---|
   | http://localhost:8080 | Aplicación web (entra con `ADMIN_USERNAME` / `ADMIN_PASSWORD`) |
   | http://localhost:8000/docs | Documentación de la API (Swagger) |
   | http://localhost:9001 | Consola de MinIO (`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`) |

## Comandos habituales

```bash
docker compose up --build -d     # reconstruir y reiniciar tras cambiar código en cualquier repositorio
docker compose logs -f worker-rq # seguir los logs de un servicio (aquí, el procesamiento de vídeos)
docker compose down              # parar todo, conservando los datos
docker compose down -v           # parar todo y BORRAR los volúmenes (BD, cola y ficheros)
```

En [`usage.md`](usage.md) hay comandos para abrir una shell en los contenedores (psql, redis-cli…).

## Problemas frecuentes

- **Todos los vídeos acaban con estado `error`.** Mira `docker compose logs worker-rq`. Lo más habitual es que falten los modelos o que sus rutas no coincidan con las variables `*_MODEL_PATH`.
- **El backend o el worker no arrancan.** Mira `docker compose logs backend-migrate`: si la migración o el seed fallan (por ejemplo, porque una variable del `.env` está vacía), no se arrancan.
- **"port is already allocated".** Otro programa usa uno de los puertos. Páralo, o cambia el lado del host del mapeo en `compose.yaml` (p. ej. `"5433:5432"`).
- **Los cambios de código no aparecen.** Hay que reconstruir las imágenes: `docker compose up --build -d`.

## Seguridad

Esta configuración está pensada para desarrollo local y demos: todos los puertos se publican en el host y los servicios usan las credenciales del `.env`. No la expongas a Internet tal cual. No subas nunca el fichero `.env` (está en `.gitignore`).

## Licencia

© 2026 Aitor Alonso Coloma. Con licencia [Creative Commons Atribución-NoComercial 4.0 Internacional (CC BY-NC 4.0)](LICENSE): puedes usarlo, compartirlo y adaptarlo con fines no comerciales, siempre que cites al autor y enlaces la licencia. Los modelos entrenados y las imágenes usadas para entrenarlos no están cubiertos por esta licencia (no se incluyen).
