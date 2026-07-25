# First-run database import

Drop your MySQL dump file(s) in this directory — `.sql`, `.sql.gz`, or `.sh`.

They are imported **only on first run**, i.e. when the `mysql-data` volume is
empty. Files run in alphabetical order, against the database named by
`MYSQL_DATABASE` in `.env`.

Notes:

- If your dump was created with `mysqldump --databases` (contains
  `CREATE DATABASE` / `USE` statements), set `MYSQL_DATABASE` in `.env` to that
  same database name so the `webapp` user is granted access to it.
- To force a re-import, wipe the data volume and start fresh:

  ```sh
  docker compose down -v
  docker compose up -d
  ```
