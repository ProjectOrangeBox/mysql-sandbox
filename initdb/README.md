# First-run database import

Drop your MySQL dump file(s) in this directory — `.sql`, `.sql.gz`, or `.sh`.

They are imported **only on first run**, i.e. when the `mysql-data` volume is
empty. Files run in alphabetical order, against the database named by
`MYSQL_DATABASE` in `.env`.

## What is here

Alphabetical order is the only ordering MySQL's entrypoint offers, so the files
are numbered to make the order deliberate rather than a coincidence of naming.
Some of it genuinely matters: a role has to exist before a permission can be
granted to it, and the ACL tables have to exist before either.

| File | What it is |
| --- | --- |
| `10-acl-tables.sql` | The six [`orange/acl`](https://github.com/ProjectOrangeBox/acl) tables, in dependency order |
| `20-acl-seed.sql` | An admin and a guest user, the role, and the orders permissions |
| `30-records.sql` | The `records` table behind the REST + Vue example |
| `35-records-seed.sql` | Three sample records |
| `40-calendar.sql` | The `calendar_events` table |
| `50-orders.sql` | Customers, orders and order lines |
| `55-orders-seed.sql` | Two orders with line items |

Leave gaps in the numbering. Inserting something between two existing steps is
otherwise a rename of everything after it.

**The guest user in `20-acl-seed.sql` is not decoration.** `orange/acl` resolves
every request without a login to the id in its `guest user` config (2 by
default), so if that row is missing, every anonymous request fails rather than
simply being unprivileged.

The admin logs in with `admin@example.com` / `orange123`. That is a published
example credential in a public repository — it is not a secret, and nothing
reachable should ever be running it.

## Notes

- If your dump was created with `mysqldump --databases` (contains
  `CREATE DATABASE` / `USE` statements), set `MYSQL_DATABASE` in `.env` to that
  same database name so the `webapp` user is granted access to it.
- To force a re-import, wipe the data volume and start fresh:

  ```sh
  docker compose down -v
  docker compose up -d
  ```

## These files are generated

They are **not** hand-maintained. They are dumped from the Phinx migrations in
[Orange-Application](https://github.com/ProjectOrangeBox/Orange-Application)
(`database/migrations` and `database/seeds`) by `composer db:export`, which
builds a scratch database, migrates and seeds it, dumps the result here, and
drops it again.

Edit a migration, not these files - anything changed here is overwritten the
next time someone regenerates. `composer db:check` fails when the two disagree.

If you are loading this into your own MySQL rather than using the sandbox, the
files are plain SQL and load in numeric order:

```sh
for f in initdb/*.sql; do mysql yourdb < "$f"; done
```

Schema and seed data are separate files on purpose, so you can take the schema
without the example rows - skip the `*-seed.sql` ones.
