# MySQL for WebApp

Dockerized MySQL 8.4 server for the webapp project (`../webapp`). Imports a
MySQL dump automatically on first run and persists data in a named volume.

## Quick start

```sh
cp env.sample .env        # then edit MYSQL_ROOT_PASSWORD
cp /path/to/dump.sql initdb/   # optional — seeds the DB on first run
docker compose up -d
```

MySQL is then available on `localhost:3306` (configurable via `MYSQL_PORT`).

## Configuration

All settings live in `.env` (see [env.sample](env.sample)):

| Variable              | Default           | Purpose                            |
| --------------------- | ----------------- | ---------------------------------- |
| `MYSQL_ROOT_PASSWORD` | `root_change_me`  | Root password — change it          |
| `MYSQL_DATABASE`      | `webapp`          | Database created on first run      |
| `MYSQL_USER`          | `webapp`          | App user (full grants on database) |
| `MYSQL_PASSWORD`      | `webapp_password` | App user password                  |
| `MYSQL_PORT`          | `3306`            | Host port to publish               |

## First-run dump import

Files in [initdb/](initdb/) (`.sql`, `.sql.gz`, `.sh`) are executed in
alphabetical order **only when the data volume is empty** — i.e. the first
`docker compose up`. Once the volume exists, they are skipped entirely.

- Plain dumps (no `CREATE DATABASE`) are imported into `MYSQL_DATABASE`.
- Dumps made with `mysqldump --databases` create their own database — set
  `MYSQL_DATABASE` in `.env` to the same name so the app user gets grants on it.

To re-import from scratch (destroys all data):

```sh
docker compose down -v
docker compose up -d
```

## Connecting from the webapp

Credentials are mirrored in the webapp's `env.sample` (`[db]` section):

- host: `host.docker.internal` from inside the webapp container,
  `127.0.0.1` from the host / CLI scripts
- PDO DSN: `mysql:host=host.docker.internal;port=3306;dbname=webapp;charset=utf8mb4`

```php
$pdo = new PDO(
    "mysql:host={$db['host']};port={$db['port']};dbname={$db['database']};charset={$db['charset']}",
    $db['username'],
    $db['password'],
    [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
);
```

## Common operations

```sh
docker compose up -d                # start
docker compose down                 # stop (data kept)
docker compose down -v              # stop and wipe data
docker compose logs -f mysql        # follow server logs
docker exec -it webapp-mysql mysql -uwebapp -pwebapp_password webapp   # SQL shell
```

The container reports `healthy` (via `mysqladmin ping`) once the server is
ready to accept connections — check with `docker compose ps`.

## Layout

```
docker-compose.yml   service definition (mysql:8.4)
env.sample           template for .env
initdb/              dump files imported on first run
```
