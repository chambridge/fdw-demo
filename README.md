# fdw-demo
Create a demo for how foreign data wrapper works in postgresql

# Getting Started

Use Docker compose launch the two databases.

```
docker-compose up
```

Next download the Netfilx movie database content:
```
wget https://raw.githubusercontent.com/neondatabase/postgres-sample-dbs/main/netflix.sql
```

Add the data into the netflix database:
```
psql -d "postgres://netflix:netflix@localhost:15432/netflix" -f netflix.sql
```

Now download the Pagila database content:
```
wget https://raw.githubusercontent.com/neondatabase/postgres-sample-dbs/main/pagila.sql
```

Populate the pagila databse:
```
psql -d "postgres://pagila:pagila@localhost:25432/pagila" -f pagila.sql
```

[How to Set Up a Foreign Data Wrapper in PostgreSQL](https://towardsdatascience.com/how-to-set-up-a-foreign-data-wrapper-in-postgresql-ebec152827f3)


The Pagila database has actor content that we want to correlate with the Netfilx data, so we want to make it a accessible to the Netflix database.

First we start with creating a read-only user in the Pagila database:
```
psql -d "postgres://pagila:pagila@localhost:25432/pagila" -c "CREATE USER fdwUser WITH PASSWORD 'secret'"
```

Grant access to the public schema:
```
psql -d "postgres://pagila:pagila@localhost:25432/pagila" -c  "GRANT USAGE ON SCHEMA PUBLIC TO fdwUser"
```

Grant access to the actor table:
```
psql -d "postgres://pagila:pagila@localhost:25432/pagila" -c  "GRANT SELECT ON actor TO fdwUser"
```

Now make sure the Foreign Data Wrapper extension is available in the Netflix DB:
```
psql -d "postgres://netflix:netflix@localhost:15432/netflix" -c "CREATE EXTENSION IF NOT EXISTS postgres_fdw"
```

Create the foreign server:
```
psql -d "postgres://netflix:netflix@localhost:15432/netflix" -c "CREATE SERVER pagila_fdw FOREIGN DATA WRAPPER postgres_fdw OPTIONS (host 'pagila_db', port '5432', dbname 'pagila')"
```

Create the user mapping:
```
psql -d "postgres://netflix:netflix@localhost:15432/netflix" -c "CREATE USER MAPPING FOR netflix SERVER pagila_fdw OPTIONS (user 'fdwuser', password 'secret')"
```

Grant access to foreign server:
```
psql -d "postgres://netflix:netflix@localhost:15432/netflix" -c "GRANT USAGE ON FOREIGN SERVER pagila_fdw TO netflix"
```

Import actor table to schema:
```
psql -d "postgres://netflix:netflix@localhost:15432/netflix" -c "IMPORT FOREIGN SCHEMA public LIMIT TO (actor) FROM SERVER pagila_fdw INTO public"
```

Now query the actor table from the netflix database:
```
psql -d "postgres://netflix:netflix@localhost:15432/netflix" -c "SELECT distinct netflix_shows.title, actor.first_name, actor.last_name, actor.actor_id  FROM netflix_shows JOIN actor ON netflix_shows.cast_members ILIKE '%' || actor.first_name || ' ' || actor.last_name || '%'"
```

Which should result in:
```
        title        | first_name  | last_name | actor_id
---------------------+-------------+-----------+----------
 The Whole Truth     | CHRISTOPHER | BERRY     |       91
 1BR                 | SUSAN       | DAVIS     |      110
 1BR                 | SUSAN       | DAVIS     |      101
 Free State of Jones | CHRISTOPHER | BERRY     |       91
(4 rows)
```

You have just queried data from a foreign database table and joined it with a local database table.
