# MySQL Sandbox

A disposable MySQL 8.4 container for local development.

It exists to give an application a real database to talk to without installing
MySQL on your machine. Unlike its cache siblings it **does persist** — in a
named volume — because a database that forgets its schema on every restart is
no use to anyone.

It ships a sample dump that seeds the example data behind the Orange
[webapp](https://github.com/ProjectOrangeBox/Orange-Application)'s REST API and
the [Vue front end](https://github.com/ProjectOrangeBox/Orange-Vue-Application)
that consumes it — see [below](#backing-the-rest-api-and-vue-example). Drop in
your own dump instead and it is a generic MySQL sandbox.

> **Development only.** The credentials in [env.sample](env.sample) are public
> and weak, and port `3306` is published to the host. Change them, or do not run
> this anywhere reachable.

Its siblings are
[redis-sandbox](https://github.com/ProjectOrangeBox/redis-sandbox) and
[memcached-sandbox](https://github.com/ProjectOrangeBox/memcached-sandbox).

## Quick start

```sh
cp env.sample .env             # then edit MYSQL_ROOT_PASSWORD
cp /path/to/dump.sql initdb/   # optional — seeds the DB on first run
docker compose up -d
```

MySQL is then available on `localhost:3306` (configurable via `MYSQL_PORT`).
The container reports `healthy` once it is accepting connections:

```sh
docker compose ps
```

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

That pair is also the quickest way back to known-good demo data after you have
poked at it — the whole point of a sandbox.

## Backing the REST API and Vue example

The bundled [initdb/webapp_sample.sql](initdb/webapp_sample.sql) creates one
table, `records`, and seeds it with three rows. That table is the far end of a
chain that runs through both example applications:

```
initdb/webapp_sample.sql          this container, imported on first run
  └── records table
        └── api/models/RecordModel.php          PDO queries
              └── api/controllers/RestController.php    JSON endpoints
                    └── src/stores/records.ts    the Vue app's Pinia store
```

`RestController` exposes plain CRUD over that one table:

| Method   | Route              | Does                     |
| -------- | ------------------ | ------------------------ |
| `GET`    | `/api/index`       | every record             |
| `GET`    | `/api/read/{id}`   | one record               |
| `POST`   | `/api/create`      | insert, body is the row  |
| `PUT`    | `/api/update/{id}` | update                   |
| `DELETE` | `/api/delete/{id}` | delete                   |

The Vue front end talks to exactly those five, via `VITE_API_BASE_URL` in its
`.env` (`http://localhost:8080/api` against the webapp's HTTP port, or
`https://localhost:8443/api` for TLS). With all three containers up you get the
whole stack: Vue on `:3000`, the API on `:8080`, this database on `:3306`.

So the seeded `records` rows are what make the example list render with
something in it on a fresh checkout, rather than an empty table and a demo that
looks broken.

**The calendar example needs one more table.** `CalendarController`
(`/api/calendar/...`) reads a `calendar_events` table that this dump does *not*
create — it ships in the webapp repo as `database/calendar_events.sql`. Apply it
yourself if you want that endpoint to work:

```sh
docker exec -i webapp-mysql mysql -uwebapp -pwebapp_password webapp \
  < /path/to/webapp/database/calendar_events.sql
```

## Connecting to it

**The host address depends on where the client is running.** The container
publishes its port to the host, so:

- from **inside an app container** — `host.docker.internal`
- from the **host** (a CLI script, a GUI client, a local `php -S`) —
  `127.0.0.1`

That is the same split the cache sandboxes use. For the webapp these values are
mirrored in its `env.sample` under `[db]`:

```
mysql:host=host.docker.internal;port=3306;dbname=webapp;charset=utf8mb4
```

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

# is the seed data there?
docker exec -it webapp-mysql mysql -uwebapp -pwebapp_password webapp \
  -e 'select id, name from records;'
```

Any MySQL GUI works too — point it at `127.0.0.1:3306` with the `.env`
credentials.

## Requirements

Docker. To talk to it from PHP you also need the `pdo_mysql` extension, which
most PHP builds ship with — the `docker exec` commands above work regardless.

## Layout

```
docker-compose.yml   service definition (mysql:8.4)
env.sample           template for .env
initdb/              dump files imported on first run
```

## License

MIT — see [LICENSE](LICENSE).
