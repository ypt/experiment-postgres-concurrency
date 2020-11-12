# Cursors and pagination stability amidst concurrent operations
*Question*: With "read committed" isolation, does paginating through query results
via a cursor yield a stable resultset amidst concurrent insertions, updates, and
deletes from other transactions?

*Answer*: Yes. A cursor under the "read committed" isolation level remains
stable when paginated amidst concurrent inserts, updates, and deletes. However,
be careful with long running transactions (for example: long running cursor
transactions to build reports), as those can block concurrent `ALTER TABLE`
operations.

See below for hands on examples:

## Setup
Bring up Postgres
```sh
docker run --rm --name my-postgres -e POSTGRES_PASSWORD=experiment -e POSTGRES_USER=experiment -p 5432:5432 postgres:11
```

Log into Postgres
```sh
psql -h localhost -p 5432 -U experiment
# when prompted, enter the password: experiment
```

Log into Postgres again in another session
```sh
psql -h localhost -p 5432 -U experiment
# when prompted, enter the password: experiment
```


## What if insert operations are happening concurrently elsewhere?
In session-1

Perform some initial setup
```sql
-- session-1
DROP TABLE events;
CREATE TABLE events (
  seq BIGSERIAL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload VARCHAR NOT NULL
);
INSERT INTO events (payload) VALUES('hello-1');
INSERT INTO events (payload) VALUES('hello-2');
```

Create a cursor and start reading it, one row at a time
```sql
-- session-1
BEGIN;
DECLARE cur CURSOR FOR SELECT * FROM events;
FETCH FORWARD 1 FROM cur;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-11-12 19:44:42.559288+00 | hello-1
-- (1 row)
FETCH FORWARD 1 FROM cur;
--  seq |              ts              | payload
-- -----+------------------------------+---------
--    2 | 2020-11-12 19:44:43.27878+00 | hello-2
-- (1 row)
```

In session-2

Insert some data concurrently
```sql
-- session-2
INSERT INTO events (payload) VALUES('hello-3');
```

In session-1

Read the next chunk from the cursor
```sql
-- session-1
FETCH FORWARD 1 FROM cur;
--  seq | ts | payload
-- -----+----+---------
-- (0 rows)
CLOSE cur;
COMMIT;
```

Note that the new value inserted in session-2 was not read.

Takeaway: the "read committed" cursor paginates stably amidst committed insert
operations happening concurrently elsewhere!

## What if update operations are happening concurrently elsewhere?
In session-1

Perform some initial setup
```sql
-- session-1
DROP TABLE events;
CREATE TABLE events (
  seq BIGSERIAL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload VARCHAR NOT NULL
);
INSERT INTO events (payload) VALUES('hello-1');
INSERT INTO events (payload) VALUES('hello-2');
```

Create a cursor and start reading it, one row at a time
```sql
-- session-1
BEGIN;
DECLARE cur CURSOR FOR SELECT * FROM events;
FETCH FORWARD 1 FROM cur;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-11-12 19:53:56.076102+00 | hello-1
-- (1 row)
```

In session-2

Update all values
```sql
-- session-2
UPDATE events SET payload = 'NEW VALUE';
```

In session-1

Fetch the next chunks from the cursor
```sql
-- session-1
FETCH FORWARD 1 FROM cur;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    2 | 2020-11-12 19:53:56.593433+00 | hello-2
-- (1 row)
FETCH FORWARD 1 FROM cur;
--  seq | ts | payload
-- -----+----+---------
-- (0 rows)
CLOSE cur;
COMMIT;
```

Note that the updated values are not visible to the cursor.

Takeaway: the "read committed" cursor paginates stably amidst committed update
operations happening concurrently elsewhere!

## What if delete operations are happening concurrently elsewhere?
In session-1

Perform some initial setup
```sql
-- session-1
DROP TABLE events;
CREATE TABLE events (
  seq BIGSERIAL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload VARCHAR NOT NULL
);
INSERT INTO events (payload) VALUES('hello-1');
INSERT INTO events (payload) VALUES('hello-2');
```

Create a cursor and start reading it, one row at a time
```sql
-- session-1
BEGIN;
DECLARE cur CURSOR FOR SELECT * FROM events;
FETCH FORWARD 1 FROM cur;
--  seq |              ts              | payload
-- -----+------------------------------+---------
--    1 | 2020-11-12 19:59:15.75771+00 | hello-1
-- (1 row)
```

In session-2

Delete everything from the table
```sql
-- session-2
DELETE FROM events;
```

In session-1

Read the next chunks from the cursor
```sql
-- session-1
FETCH FORWARD 1 FROM cur;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    2 | 2020-11-12 19:59:16.669474+00 | hello-2
-- (1 row)
FETCH FORWARD 1 FROM cur;
--  seq | ts | payload
-- -----+----+---------
-- (0 rows)
CLOSE cur;
COMMIT;
```

Note that the deleted rows are not yet deleted from the cursor's viewpoint.

Takeaway: the "read committed" cursor paginates stably amidst committed delete
operations happening concurrently elsewhere!

## What if table schema changes are happening concurrently elsewhere?

In session-1

Perform some initial setup
```sql
-- session-1
DROP TABLE events;
CREATE TABLE events (
  seq BIGSERIAL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload VARCHAR NOT NULL
);
INSERT INTO events (payload) VALUES('hello-1');
INSERT INTO events (payload) VALUES('hello-2');
```

Create a cursor and start reading it, one row at a time
```sql
-- session-1
BEGIN;
DECLARE cur CURSOR FOR SELECT * FROM events;
FETCH FORWARD 1 FROM cur;
--  seq |              ts              | payload
-- -----+------------------------------+---------
--    1 | 2020-11-12 19:59:15.75771+00 | hello-1
-- (1 row)
```

In session-2

Change the table schema
```sql
-- session-2
BEGIN;
ALTER TABLE events ADD COLUMN newcolumn varchar(30);
ALTER TABLE events DROP COLUMN ts;
ALTER TABLE events ALTER COLUMN seq TYPE varchar(80);
COMMIT;
```
Note that this operation doesn't complete, and hangs.

In session-1

Read the next chunks from the cursor
```sql
-- session-1
FETCH FORWARD 1 FROM cur;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    2 | 2020-11-12 19:59:16.669474+00 | hello-2
-- (1 row)
FETCH FORWARD 1 FROM cur;
--  seq | ts | payload
-- -----+----+---------
-- (0 rows)
CLOSE cur;
COMMIT;
```

Note that session-2 _now_ executes and finishes.

Takeaway: the cursor transaction blocks table schema changes.

## Generalizing concurrent behavior for table schema changes
In fact, the above behavior can be generalized for table schema changes and
transactions.

Perform some initial setup
```sql
-- session-1
DROP TABLE events;
CREATE TABLE events (
  seq BIGSERIAL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload VARCHAR NOT NULL
);
INSERT INTO events (payload) VALUES('hello-1');
INSERT INTO events (payload) VALUES('hello-2');
```

In session-1

Start a transaction
```sql
-- session-1
BEGIN;
SELECT * from events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-11-12 20:33:52.937686+00 | hello-1
--    2 | 2020-11-12 20:33:54.056751+00 | hello-2
-- (2 rows)
```

In session-2

Start a transaction and table schema change operation
```sql
-- session-2
BEGIN;
ALTER TABLE events ADD COLUMN newcolumn varchar(30);
```
Note that the above hangs.

In session-1

Now close the transaction
```sql
-- session-1
COMMIT;
```

Note that the `ALTER TABLE` in session-2 now executes once session-1's transaction was
committed.

In session-2

Finally commit the transaction
```sql
-- session-2
COMMIT;
```
