# Container access

Run these commands from this folder (where `compose.yaml` is). `docker compose exec` uses the service names, so they work whatever the container names are. The credentials are read from the container's own environment, so there is nothing to replace.

## PostgreSQL

```bash
# psql shell
docker compose exec db-postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'

# plain shell
docker compose exec db-postgres sh
```

Useful psql commands:

```sql
\dt                         -- list tables
\d videos                   -- describe a table
SELECT * FROM video_status; -- video statuses created by the seed
\q                          -- quit
```

## Redis

```bash
docker compose exec cache-redis sh -c 'redis-cli -a "$REDIS_PASSWORD"'
```

Inside `redis-cli`, `KEYS rq:*` lists the RQ queues and jobs.

## MinIO

```bash
docker compose exec filesystem-minio sh
```

The web console (http://localhost:9001) is usually easier for browsing the bucket.

## Backend and worker

```bash
docker compose exec backend-fastapi sh
docker compose exec worker-rq sh

# run the migrations and the seed again (they are idempotent)
docker compose run --rm backend-migrate
```

## Logs and status

```bash
docker compose ps                     # services and their status
docker compose logs -f backend-fastapi
docker compose logs backend-migrate   # migration and seed output
docker compose logs -f worker-rq      # video processing
```
