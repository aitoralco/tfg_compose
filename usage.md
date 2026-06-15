# Container Access — podman exec

## PostgreSQL (most useful)

```bash
# Open psql shell directly
podman exec -it tfg_dockercompose-db-postgres-1 psql -U $DB_USER -d $DB_NAME

# Or get a bash shell first
podman exec -it tfg_dockercompose-db-postgres-1 bash
```

### Useful psql commands once inside

```sql
\l                  -- list databases
\c <db_name>        -- connect to a database
\dt                 -- list tables
\d <table>          -- describe a table
\q                  -- quit
```

## Redis

```bash
podman exec -it tfg_dockercompose-cache-redis-1 redis-cli -a $REDIS_PASSWORD
```

## MinIO

```bash
podman exec -it tfg_dockercompose-filesystem-minio-1 bash
```

---

## Find the exact container name

If the name above doesn't match, list running containers:

```bash
podman ps
```