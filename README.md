# Whaler — Docker Compose

**English** | [Español](README.es.md)

This repository starts the whole **Whaler** stack with Docker Compose. Whaler is a web app for uploading cetacean videos and processing them with AI: it detects the whales, crops them, detects their marks and classifies the species. The frontend, backend and worker live in their own repositories. This one only contains the `compose.yaml` that builds them and connects them to the data services (PostgreSQL, Redis and MinIO).

## Repositories

| Repository | Folder | What it is |
|---|---|---|
| [aitoralco/tfg_frontend](https://github.com/aitoralco/tfg_frontend) | `tfg_frontend` | Web client (Angular + Nginx) |
| [aitoralco/tfg_backend](https://github.com/aitoralco/tfg_backend) | `tfg_backend` | REST API (FastAPI), DB migrations and seed |
| [aitoralco/tfg_worker](https://github.com/aitoralco/tfg_worker) | `tfg_worker` | AI processing worker (RQ + YOLO) |
| [aitoralco/tfg_compose](https://github.com/aitoralco/tfg_compose) | `tfg_dockercompose` (any name) | This repository |

`compose.yaml` builds the images from `../tfg_frontend`, `../tfg_backend` and `../tfg_worker`. **The four repositories have to be cloned side by side in the same folder, with those exact folder names**:

```
code/
├── tfg_backend/
├── tfg_frontend/
├── tfg_worker/
└── tfg_dockercompose/   # this repository
```

## Requirements

- **Docker** with **Compose v2** (the `docker compose` command): Docker Desktop on Windows/macOS, or Docker Engine + the Compose plugin on Linux.
- **Git**, to clone the four repositories.
- **The trained YOLO models for the worker** (see below). Without them the stack starts, but no video can be processed.
- **Free ports** on the host: `8080` (frontend), `8000` (backend), `5432` (PostgreSQL), `6379` (Redis), `9000` and `9001` (MinIO).
- **Disk space and memory.** The worker image is the largest one: it includes PyTorch (CPU build), OpenCV and ffmpeg. The videos and their results are stored in Docker volumes.
- No GPU is needed: the worker runs on CPU (processing is slow).

### Worker models

> **The trained model weights are not included in any repository.** They were trained with images that do not belong to the author: the images were provided only for this final degree project (TFG), so neither they nor the models trained with them can be distributed.

To process videos, place your own weights in the worker repository before building:

```
tfg_worker/app/ai/trained_models/
├── detect_whales/best.pt     # cetacean detection (OBB)      → DW_MODEL_PATH
├── detect_marks/best.pt      # mark detection                → DEM_MODEL_PATH
└── classify_whales/best.pt   # species classification        → CW_MODEL_PATH
```

The worker image copies this folder (it is ignored by git, not by Docker). The `*_MODEL_PATH` variables in `.env` point to these files inside the container (`/worker/app/ai/...`). If a model is missing, every uploaded video ends with status `error` in the DW phase. More details in the [worker README](https://github.com/aitoralco/tfg_worker#models).

## Architecture

| Service | Image / build | Host port | Role |
|---|---|---|---|
| `frontend-angular` | `../tfg_frontend` | `8080` | Serves the Angular app with Nginx |
| `backend-fastapi` | `../tfg_backend` | `8000` | REST API: users, video upload and streaming, AI results |
| `backend-migrate` | `../tfg_backend` | — | One-off job: creates the DB schema and the base data, then exits |
| `worker-rq` | `../tfg_worker` | — | Takes jobs from Redis and processes the videos with the YOLO models |
| `db-postgres` | `postgres:15-alpine` | `5432` | Relational DB: users, roles, videos and their status |
| `cache-redis` | `redis:latest` | `6379` | Job queue (RQ) between the backend and the worker |
| `filesystem-minio` | `quay.io/minio/minio:latest` | `9000` / `9001` | S3-compatible object storage: videos, thumbnails and AI results (API / web console) |

```
                         browser
       ┌───────────────────┼─────────────────────────┐
 :8080 │ app         :8000 │ API, video stream  :9000 │ presigned URLs
       ▼                   ▼                         ▼
 frontend-angular    backend-fastapi ──────────▶ filesystem-minio
                       │         │   files           ▲
                   SQL │         │ enqueue job       │ video in, results out
                       ▼         ▼                   │
                 db-postgres   cache-redis ──job─▶ worker-rq
                       ▲                             │
                       └────── video status ─────────┘
```

Processing flow:

1. The user uploads a video from the frontend. The backend stores it in MinIO, creates the row in PostgreSQL and enqueues `process_video(video_id)` in Redis.
2. The worker takes the job, downloads the video from MinIO and runs the phases DW → EW → DEM → CW. It updates the video status in the DB at each phase and uploads the results to MinIO.
3. The frontend asks the backend for the status and the results (annotated video, crops, classification).

### Why it starts this way

The services start in this order, controlled by `depends_on` and healthchecks:

```
db-postgres (healthy) ──▶ backend-migrate (exits with code 0) ──┬──▶ backend-fastapi ──▶ frontend-angular
cache-redis (healthy) ──────────────────────────────────────────┤
filesystem-minio (healthy) ─────────────────────────────────────┴──▶ worker-rq
```

- **Healthchecks, not just "started".** A container can be running before its service accepts connections. PostgreSQL (`pg_isready`), Redis (`redis-cli ping`) and MinIO (`/minio/health/live`) have healthchecks, and the services that use them wait for `service_healthy`.
- **A separate migration job.** `backend-migrate` uses the backend image but runs `alembic upgrade head` (creates or updates the schema) and then `python -m app.db.starter` (seed: roles, video statuses and the admin user from `ADMIN_*`), and exits. The backend and the worker wait for it to finish successfully (`service_completed_successfully`). This way:
  - the tables exist before anything queries them,
  - the migrations run only once, not in every replica of the backend,
  - the worker finds the video statuses, which it looks up by name (`processing_dw`, `processed`…),
  - if a migration fails, the backend and the worker do not start, and the error is in `docker compose logs backend-migrate`.
- **Shared configuration.** The backend and the migration job use the same environment variables (the `x-backend-env` block), so they always connect to the same DB.
- **The bucket is created by the backend.** If `BUCKET_NAME` does not exist in MinIO, the backend creates it the first time it connects.
- **Two MinIO URLs.** Inside Docker the backend and the worker reach MinIO as `http://filesystem-minio:9000`, but that hostname does not exist for the browser. That is why the backend signs the file URLs with `FILESYSTEM_PUBLIC_URL` (`http://localhost:9000`).
- **The frontend calls the backend from the browser**, not from inside Docker. Its API URL (`http://127.0.0.1:8000`) is fixed when the image is built. To serve it from another host, change `tfg_frontend/src/environments/environment.ts`, add the origin to the backend CORS list and rebuild.

### Persistence

Data is kept in named Docker volumes, so it survives `docker compose down`:

| Volume | Service | Content |
|---|---|---|
| `postgres_data` | `db-postgres` | Database |
| `redis_data` | `cache-redis` | Redis data (queue) |
| `minio_data` | `filesystem-minio` | Videos, thumbnails and AI results |

Redis loads its configuration from `redis/redis.conf`. The password is not in that file: it is passed with `--requirepass ${REDIS_PASSWORD}`.

## Getting started

1. **Clone the repositories** side by side:
   ```bash
   mkdir whaler && cd whaler
   git clone https://github.com/aitoralco/tfg_backend.git
   git clone https://github.com/aitoralco/tfg_frontend.git
   git clone https://github.com/aitoralco/tfg_worker.git
   git clone https://github.com/aitoralco/tfg_compose.git tfg_dockercompose
   cd tfg_dockercompose
   ```
2. **Add the models** in `../tfg_worker/app/ai/trained_models/` (see [Worker models](#worker-models)).
3. **Create the `.env`** from the example and fill in the values:
   ```bash
   cp .env.example .env
   ```

   | Variable | Use |
   |---|---|
   | `DB_NAME`, `DB_USER`, `DB_PASSWORD` | PostgreSQL database and credentials |
   | `ADMIN_USERNAME`, `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Admin user created by the seed (password stored hashed) |
   | `REDIS_PASSWORD` | Redis password |
   | `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD` | MinIO credentials (the password needs at least 8 characters) |
   | `BUCKET_NAME` | Bucket where everything is stored (e.g. `videos-bucket`) |
   | `JWT_SECRET_KEY` | Key used to sign the login tokens (a long random string) |
   | `DW_MODEL_PATH`, `DEM_MODEL_PATH`, `CW_MODEL_PATH` | Model paths inside the worker container (the defaults are fine) |

4. **Build and start** everything:
   ```bash
   docker compose up --build -d
   ```
   The first build takes a while (mainly the worker, because of PyTorch).
5. **Check the status:**
   ```bash
   docker compose ps
   ```
   All services should be `running`, and the healthchecks `healthy`. `backend-migrate` appears as `exited (0)`: that is expected, it only runs once at startup.
6. **Open the app:**

   | URL | What |
   |---|---|
   | http://localhost:8080 | Web app (log in with `ADMIN_USERNAME` / `ADMIN_PASSWORD`) |
   | http://localhost:8000/docs | API documentation (Swagger) |
   | http://localhost:9001 | MinIO console (`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`) |

## Everyday commands

```bash
docker compose up --build -d     # rebuild and restart after changing code in any repository
docker compose logs -f worker-rq # follow a service's logs (here, the video processing)
docker compose down              # stop everything, keeping the data
docker compose down -v           # stop everything and DELETE the volumes (DB, queue and files)
```

[`usage.md`](usage.md) has commands to open a shell in the containers (psql, redis-cli…).

## Troubleshooting

- **Every video ends with status `error`.** Check `docker compose logs worker-rq`. The most common cause is that the models are missing or their paths do not match the `*_MODEL_PATH` variables.
- **The backend or the worker does not start.** Look at `docker compose logs backend-migrate`: if the migration or the seed fails (for example, because a variable in `.env` is empty), they are not started.
- **"port is already allocated".** Another program is using one of the ports. Stop it, or change the host side of the mapping in `compose.yaml` (e.g. `"5433:5432"`).
- **Code changes do not show up.** The images have to be rebuilt: `docker compose up --build -d`.

## Security

This setup is meant for local development and demos: all the ports are published on the host and the services use the credentials from `.env`. Do not expose it to the Internet as is. Never commit the `.env` file (it is in `.gitignore`).

## License

© 2026 Aitor Alonso Coloma. Licensed under the [MIT License](LICENSE). The trained models and the images used to train them are not covered by this license (they are not included).
