# Coordinating snapshot reads with event sequences

## Context
Imagine...

- Your application is a traditional stateless service + RDBMS combo. It does
  _not_ have an event sourced architecture.
- Your application has just started publishing a new change event to a
  distributed log (i.e. Kafka, Pulsar) for consumption by other services.
- Consumer services want to leverage the new change event for [event-carried
  state
  transfer](https://martinfowler.com/articles/201701-event-driven.html#Event-carriedStateTransfer),
  but require old (pre change event) data to be backfilled into their service.

The big question...

- How do we backfill the old (pre change event) data into a consuming service?

At a high level, here are some pieces of the puzzle to solve...

1. Challenge 1: How can we deliver a _consistent_ snapshot of the old data
   delivered to the consuming service. For example, paginated REST API datasets
   might shift around underneath during pagination.
2. Challenge 2: The consuming service needs to know _where_ in the event
   sequence that snapshot of data was taken. This is so that the consumer can
   process events, then switch over to processing the snapshot at the
   appropriate point, and then switch back to processing events.

    ```
    "Hey! FYI, the snapshot was taken here in the message sequence chronology!"
                    |
                    v
    msg-1 msg-2 ...   ... msg-n
    ```

A data layer solution, change data capture, via
[Debezium](https://debezium.io/documentation/reference/connectors/postgresql.html)
implements a similar idea via constructs available through [logical replication
slots](https://www.postgresql.org/docs/11/protocol-replication.html#PROTOCOL-REPLICATION-CREATE-SLOT):
log sequence numbers, and snapshots.

What can we do if we don't leverage a _data layer_ solution such as Debezium and
instead want a solution at the _application layer_?

Answers, at a high level...

1. Re: Challenge 1: "repeatable read" transaction isolation, or a [single query + cursor combo](cursors_and_pagination_stability.md)
   both provide multiple stable reads at a snapshot in time.
2. Re: Challenge 2: some sort of coordination will be required to coordinate the
   concurrent processes writing to the distributed log and the process that
   creates the snapshot in order to properly note _where_ in the distributed log
   event sequence the snapshot occurs.

Let's see what sort of tools we can use to tackle this...

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

Log into Postgres again in yet another session
```sh
psql -h localhost -p 5432 -U experiment
# when prompted, enter the password: experiment
```

In session-1

```sql
-- session-1

DROP TABLE mytable;
CREATE TABLE mytable (
  seq BIGSERIAL,
  ts TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload VARCHAR NOT NULL
);
INSERT INTO mytable (payload) VALUES('hello-1');
INSERT INTO mytable (payload) VALUES('hello-2');
```

## Hands on example

In session-1

- Lock the rows/tables being that we will later query. This is to prevent
  modifications to it while we're noting where our query falls within our
  external distributed log's (i.e. Kafka, Pulsar, etc.) sequence of messages.
  Ultimately, the goal of this is to block writes to the relevant
  topic/partition of the distributed log while the snapshot position is being
  noted.
- export the snapshot - Via
  [pg_export_snapshot](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-SNAPSHOT-SYNCHRONIZATION).
  This is the Postgres snapshot that we will use to execute our consistent read.


```sql
-- session-1

BEGIN;

-- set isolation level to "repeatable read"
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- lock table
-- TODO: see if other lock modes are usable.
-- https://www.postgresql.org/docs/11/explicit-locking.html
LOCK TABLE mytable IN ACCESS EXCLUSIVE MODE;

-- export this transaction's snapshot for use elsewhere
SELECT pg_export_snapshot();
--  pg_export_snapshot
-- ---------------------
--  00000003-00000006-1
-- (1 row)
```

In session-3

- confirm that writes to the table are blocked. Blocking writes to this table
  blocks operations that affect it and thus effectively blocks writes to the
  relevant distributed log topic/partition.

```sql
-- session-3

INSERT INTO mytable (payload) VALUES('hello-3');
-- this should block
```

In your app

- Now that writes to the relevant partition of your distributed log are blocked,
  do something to indicate "tthe snapshot was taken here in the message sequence
  chronology". For example, send a marker message to the relevant distributed
  log topic.

In session-2

- import snapshot

```sql
-- session-2

BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION SNAPSHOT '00000003-00000006-1'; -- this should be the value from our earlier execution of pg_export_snapshot()
```

In session -1

- end the transaction to release the lock and reopen the gates for messages to
  flow to the distributed log.

```sql
-- session-1

COMMIT;
```

In session-3

- Confirm that the write operation from earlier has now finished executing, now
  that the table it needs is unlocked.
- Reading the table confirms that the new row has been inserted.

```sql
-- session-3

SELECT * from mytable;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-11-24 20:45:03.621027+00 | hello-1
--    2 | 2020-11-24 20:45:04.440505+00 | hello-2
--    3 | 2020-11-24 20:49:34.958859+00 | hello-3
-- (3 rows)
```

In session-2

- execute a query

```sql
-- session-2

SELECT * from mytable;
--  seq |              ts               | payload
-- -----+-------------------------------+---------
--    1 | 2020-11-24 20:45:03.621027+00 | hello-1
--    2 | 2020-11-24 20:45:04.440505+00 | hello-2
-- (2 rows)
```

Note that the new row from session-3 is not included here. That confirms that
we're querying using the snapshot's older view of the world.

In your app

- consume the query results, transform them as necessary into messages sent to
  the distributed log.

In session-2

- finish up

```sql
-- session-2

COMMIT;
```

## Other things to solve, but out of scope for this doc
This document outlines a solution at a high level, and there are still more
details to be worked out. For example:

- What sort of snapshot delivery protocol and transport mechanism do we use? For
  example, if a paginated REST API is unable to deliver a dataset in a
  consistent manner, what will?
- What do the backfill data records look like, exactly?
- How and where exactly do we note the metadata of "the snapshot was taken here
  in the message sequence chronology"?
- How might we handle failure cases?
