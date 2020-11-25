# Experiment with Postgres concurrency, transaction isolation levels, and ordering

- https://www.postgresql.org/docs/11/transaction-iso.html
- https://www.postgresql.org/docs/11/sql-set-transaction.html
- https://www.postgresql.org/docs/11/functions-sequence.html

## Hands on examples

### Setup
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

### Example 1 - "read committed" isolation level
The default isolation level is "[read
committed](https://www.postgresql.org/docs/11/transaction-iso.html#XACT-READ-COMMITTED)".

Create a table
```sql
CREATE TABLE events (
  seq BIGSERIAL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload VARCHAR NOT NULL
);
```

In session-1
```sql
BEGIN; -- The default isolation level is "read committed"
INSERT INTO events (payload) VALUES('hello-1');
-- DO NOT COMMIT YET

SELECT * FROM events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-10-21 14:39:43.016317+00 | hello-1
```

In session-2
```sql
BEGIN;
INSERT INTO events (payload) VALUES('hello-2');
COMMIT;

SELECT * FROM events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    2 | 2020-10-21 14:40:07.018157+00 | hello-2
```

In session-1
```sql
SELECT * FROM events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-10-21 14:39:43.016317+00 | hello-1
--    2 | 2020-10-21 14:40:07.018157+00 | hello-2

COMMIT;
```

Note: the commit from session-2 is now visible to session-1, even while
session-1 is still in a transaction that started before the transaction from
session-2.

In session-2
```sql
SELECT *, xmin FROM events;
--  seq |              ts               | payload | xmin
-- -----+-------------------------------+---------+------
--    1 | 2020-10-21 14:39:43.016317+00 | hello-1 |  571
--    2 | 2020-10-21 14:40:07.018157+00 | hello-2 |  572
```

Takeaway: the order of _when_ rows become visible externally does not
necessarily correlate to what their auto-generated `SERIAL` sequence number or
their `TIMESTAMPTZ` `DEFAULT` values indicate.

### Example 2 - "repeatable read" isolation level
Now, let's try the same as above with "[repeatable
read](https://www.postgresql.org/docs/11/transaction-iso.html#XACT-REPEATABLE-READ)"
isolation level. This is slightly stricter than "read committed".

```sql
DROP TABLE events;
CREATE TABLE events (
  seq BIGSERIAL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload VARCHAR NOT NULL
);
```

In session-1
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ; -- set isolation level to "repeatable read"
INSERT INTO events (payload) VALUES('hello-1');
-- DO NOT COMMIT YET

SELECT * FROM events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-10-21 15:26:52.774204+00 | hello-1
```

In session-2
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
INSERT INTO events (payload) VALUES('hello-2');
COMMIT;

SELECT * from events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    2 | 2020-10-21 15:26:59.251274+00 | hello-2
```

In session-1
```sql
SELECT * FROM events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-10-21 15:26:52.774204+00 | hello-1

COMMIT;
```

Note: the committed transaction from session-2 is not yet visible to session-1
above.

In session-2
```sql
SELECT *, xmin FROM events;
--  seq |              ts               | payload | xmin
-- -----+-------------------------------+---------+------
--    1 | 2020-10-21 15:26:52.774204+00 | hello-1 |  581
--    2 | 2020-10-21 15:26:59.251274+00 | hello-2 |  582
```

Takeaways: Consider "repeatable read" if you need stable reads within a
transaction.

### Example 3 - "serializable" isolation level
Now, let's try the same as above with
"[serializable](https://www.postgresql.org/docs/11/transaction-iso.html#XACT-SERIALIZABLE)"
isolation level. This is the strictest isolation level, and works similarly to
"repeatable read", except that it monitors for conditions which could make
execution of a concurrent set of serializable transactions behave in a manner
inconsistent with all possible serial (one at a time) executions of those
transactions.

```sql
DROP TABLE events;
CREATE TABLE events (
  seq BIGSERIAL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload VARCHAR NOT NULL
);
```

In session-1
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE; -- set isolation level to "serializable"
INSERT INTO events (payload) VALUES('hello-1');
-- DO NOT COMMIT YET

SELECT * FROM events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-10-21 16:51:58.291404+00 | hello-1
```

In session-2
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
INSERT INTO events (payload) VALUES('hello-2');
COMMIT;

SELECT * from events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    2 | 2020-10-21 16:52:18.329828+00 | hello-2
```

In session-1
```sql
SELECT * FROM events;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-10-21 16:51:58.291404+00 | hello-1

COMMIT;
```

Note: the committed transaction from session-2 is not yet visible to session-1
above.

In session-2
```sql
SELECT *, xmin FROM events;
--  seq |              ts               | payload | xmin
-- -----+-------------------------------+---------+------
--    1 | 2020-10-21 16:51:58.291404+00 | hello-1 |  585
--    2 | 2020-10-21 16:52:18.329828+00 | hello-2 |  586
```

Takeaways: Consider "serializable" if you need stable reads within a transaction
AND behavior consistent with executing operations one at a time serially. For
example, side-effects (writes) from one transaction that affect dependencies of
another transaction will not be allowed and result in an error returned in one
of the conflicting transactions. See the [example
here](https://www.postgresql.org/docs/11/transaction-iso.html#XACT-SERIALIZABLE).

TODO:
- add an example with "serializable" that results in a serialization anomaly
- experiment with `SERIALIZABLE READ ONLY DEFERRABLE`