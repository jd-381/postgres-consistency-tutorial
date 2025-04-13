# About
This is a Postgres tutorial on consistency using transaction snapshots.

The goal is to learng how to manage consistency for dumping a large table in parallel for replication.

# Set Up The Environment
Start the containers:
```sh
docker-compose up -d
```

There are 2 Postgres containers. `p1` is the publisher and `s1` is the subscriber.
```sh
docker ps
```

Create the test table `t1` on both `p1` and `s1`.
```sh
cat sql/create_table.sql | docker exec -i p1 psql -U postgres
cat sql/create_table.sql | docker exec -i s1 psql -U postgres
```

# Tutorial

### In Tab 1

Imitate app writes to table `t1`. This will insert 1 row every second.
```sh
while true ; do docker exec -i p1 psql -U postgres -c "insert into public.t1 (created_at) values (now())" ; sleep 1 ; done
```

### In Tab 2

Connect to `p1` in replication mode.
```sh
docker exec -it p1 psql -U postgres "dbname=postgres replication=database"
```

Run this a few times to see the row count incrementing (app writes).
```sql
SELECT COUNT(1) FROM t1;
```

Create a publication.
```sql
CREATE PUBLICATION pub_p1_to_s1 FOR TABLE t1;
```

Create a replication slot and note the `snapshot_name`.
```sql
CREATE_REPLICATION_SLOT slot_p1_to_s1 LOGICAL pgoutput EXPORT_SNAPSHOT;
   slot_name   | consistent_point |    snapshot_name    | output_plugin 
---------------+------------------+---------------------+---------------
 slot_p1_to_s1 | 0/15745E8        | 00000071-0000000B-1 | pgoutput
(1 row)
```

> This connection will be referred to as the `snapshot thread`. It has created a consistent moment in time. As long as this connection remains open, other connections can use the snapshot to access the same consistent view. 

>Connecting in replication mode interfaces with a different protocol to execute replication commands directly that are otherwise not available, such as `CREATE_REPLICATION_SLOT`. The reason for not using the more common `pg_create_logical_replication_slot()` is that it does not export a snapshot.

> Documentation: https://www.postgresql.org/docs/current/protocol-replication.html

Run this a few times to see the LSN is not advancing (because the slot is inactive). Since the slot was created in replication mode, we have a snapshot of the database consistent with this LSN. This means subscribers built from tables inside this snapshot can connect to this replication slot. Sick!
```sql
SELECT slot_name, active, restart_lsn, confirmed_flush_lsn  FROM pg_replication_slots;
   slot_name   | active | restart_lsn | confirmed_flush_lsn 
---------------+--------+-------------+---------------------
 slot_p1_to_s1 | f      | 0/15745B0   | 0/15745E8
(1 row)
```

Leave the `snapshot thread` open and keep going!

### In Tab 3

Create another connection to `p1` (not in replication mode).
```sh
docker exec -it p1 psql -U postgres
```

Run this a few times to see the row count incrementing (app writes).
```sql
SELECT COUNT(1) FROM t1;
```

Start a transaction and set the snapshot to the name from the `snapshot thread`.
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION SNAPSHOT '00000071-0000000B-1';
```

Rerun the row count query to see it is no longer incrementing and IS LOWER than the previous counts. This is the row count at the time the snapshot was created in the `snapshot thread`. Sick!
```sql
SELECT COUNT(1) FROM t1;
```

> These connections will be referred to as the `dump threads`. This is the butter for dumping a large table in chunks using parallel `dump threads` which is exponentially faster than the single-threaded `pg_dump` or `COPY` utils. This tutorial will only show case a single `dump thread` but you get the idea.

Close (`exit`) out of this `dump thread` and back to the command line (but do not close the `snapshot thread`).

Still in tab 3, create a single `dump thread` using the `snapshot_name` and dump the table `t1` from `p1` into `s1`.
```sh
docker exec -i p1 psql -U postgres -t -q -c "BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ; SET TRANSACTION SNAPSHOT '00000071-0000000B-1'; COPY (SELECT * FROM t1) TO STDOUT;" | \
docker exec -i s1 psql -U postgres -c "COPY t1 FROM STDIN;"
```

### In Tab 4

Connect to `s1` (again, not in replication mode).
```sh
docker exec -it s1 psql -U postgres
```

Run the row count query to see it is not incrementing and matches the count from the snapshot.
```sql
SELECT COUNT(1) FROM t1;
```

Time to hook up replication. Create a subscription to the slot created in the `snapshot thread`. Disable creating a new slot and copying data as these are already completed.
- As soon as the subscription connects to the slot, replication events will immediately flow despite the `snapshot thread` still being open. That's fine, that's how it works.
```sql
CREATE SUBSCRIPTION sub_p1_to_s1
CONNECTION 'host=p1 port=5432 dbname=postgres user=postgres password=password'
PUBLICATION pub_p1_to_s1
WITH (slot_name = 'slot_p1_to_s1', create_slot = false, copy_data = false);
```

Run this a few times to see the row count incrementing from replication.
```sql
SELECT COUNT(1) FROM t1;
```

Next is cleanup and consistency validation.

### In Tab 1

Cancel the app writes to validate the tables are in sync.

### In Tab 2

Close (`exit`) out of the `snapshot thread` for good measure.

### In Tab 3

Connect back to `p1`.
```sh
docker exec -it p1 psql -U postgres
```

With writes stopped, get the checksums of table `t1`.
```sql
SELECT MAX(id), MAX(created_at), COUNT(1), md5(array_agg(md5((t.*)::text))::text)
FROM (
    SELECT *
    FROM t1
    ORDER BY id
) AS t;
 max  |              max              | count |               md5                
------+-------------------------------+-------+----------------------------------
 1462 | 2025-04-13 01:23:41.353628+00 |  1462 | c364f1ca7ca012764cd73644d999aed9
(1 row)
```

Check the replication slot a few times to make sure the LSNs are not advancing. If they are, it means replication on `s1` is still catching up. Wait until the LSNs stop advancing before continuing the tutorial.
```sql
SELECT slot_name, active, restart_lsn, confirmed_flush_lsn  FROM pg_replication_slots;
```

### In Tab 4

Back on `s1`, with writes stopped, get the checksums of table `t1`. All values match `p1` indicating a consistent replicated copy.
```sql
SELECT MAX(id), MAX(created_at), COUNT(1), md5(array_agg(md5((t.*)::text))::text)
FROM (
    SELECT *
    FROM t1
    ORDER BY id
) AS t;
 max  |              max              | count |               md5                
------+-------------------------------+-------+----------------------------------
 1462 | 2025-04-13 01:23:41.353628+00 |  1462 | c364f1ca7ca012764cd73644d999aed9
(1 row)
```