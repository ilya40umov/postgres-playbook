# postgres-playbook

Recipes and notes on using PostgreSQL 

## Space Analysis 

### Get database size

```sql
-- determine size of 'mydb' database
SELECT pg_size_pretty(pg_database_size('mydb'));
```

### Get table size

```sql
-- determine size of each table in 'public' schema
WITH tables AS (
    SELECT table_schema, table_name
    FROM information_schema.tables
    WHERE table_schema = 'public'
)
SELECT table_name, 
       pg_size_pretty(pg_total_relation_size(table_name::text)) AS table_size
FROM tables
ORDER BY 2 DESC;
``` 

### List tables with pg_toast

NOTE: `pg_toast` is used by Postgres to store big column values that don't fit into a single page (e.g. a JSONB object can easily be over 8K in size).

```sql
-- list tables with pg_toast
SELECT oid::regclass,
       reltoastrelid::regclass,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_size
FROM pg_class
WHERE relkind = 'r'
  AND reltoastrelid <> 0
ORDER BY 3 DESC;
```

* [Postgres Wiki :: TOAST](https://wiki.postgresql.org/wiki/TOAST)
* [Postgres Docs :: TOAST](https://www.postgresql.org/docs/current/storage-toast.html)

### Determine sizes of individual indices

```sql
-- get index sizes
SELECT
    pg_stat_all_indexes.relname,
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
    pg_size_pretty(pg_relation_size(relid)) as table_size,
    pg_size_pretty(pg_total_relation_size(relid)) as total_relation_size,
    reltuples::bigint as table_row_count
FROM pg_stat_all_indexes JOIN pg_class ON pg_stat_all_indexes.relid = pg_class.oid
WHERE pg_stat_all_indexes.schemaname = 'public'
ORDER BY pg_total_relation_size(relid) DESC, pg_relation_size(indexrelid) DESC;
```

### Get the total size of all indices

```sql
-- get total size of indices
SELECT
    pg_size_pretty(sum(pg_relation_size(indexrelid))) as total_size_of_all_indices
FROM pg_stat_all_indexes JOIN pg_class ON pg_stat_all_indexes.relid = pg_class.oid
WHERE pg_stat_all_indexes.schemaname = 'public';
```

## Indices

### Get invalid indices

NOTE: You may end up with an "invalid" index if you try to create an index with unique constraint in background and this operation fails.

```sql
-- get invalid indices
SELECT
    *
FROM pg_class inner join pg_index ON pg_index.indexrelid = pg_class.oid
WHERE pg_index.indisvalid = false;
```

* [Problems with concurrent Postgres indexes](https://medium.com/carwow-product-engineering/problems-with-concurrent-postgres-indexes-and-how-to-solve-them-c57f7656c852)

### Random slow writes on tables with GIN indices

NOTE: Read about the problem [here](https://iamsafts.com/posts/postgres-gin-performance/)

```sql
-- find all GIN indices
SELECT pg_get_indexdef(indexrelid) from pg_index
WHERE pg_get_indexdef(indexrelid) ~ 'USING (gin )';
```

## Locks & Queries

* [Understanding PostgreSQL locks](https://shiroyasha.io/understanding-postgresql-locks.html)
* [Useful PostgreSQL Queries and Commands](https://gist.github.com/rgreenjr/3637525)
* [Postgres Wiki :: Lock Monitoring](https://wiki.postgresql.org/wiki/Lock_Monitoring)
* [Aurora :: Lock:Relation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/apg-waits.lockrelation.html)

### List not granted locks

```sql
-- list NOT granted locks
select relation::regclass, * from pg_locks where NOT granted;
```

### List granted locks

```sql
-- list granted locks
select relation::regclass, * from pg_locks where granted;
```

### List running queries

```sql
-- list running queries / transactions
SELECT pid, state, age(clock_timestamp(), query_start), usename, query
FROM pg_stat_activity
WHERE state != 'idle' AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY query_start desc;
```

### List locked queries
```sql
-- list locked queries
SELECT blocked_locks.pid         AS blocked_pid,
       blocked_activity.usename  AS blocked_user,
       blocking_locks.pid        AS blocking_pid,
       blocking_activity.usename AS blocking_user,
       blocked_activity.query    AS blocked_statement,
       blocking_activity.query   AS current_statement_in_blocking_process
FROM  pg_catalog.pg_locks blocked_locks
          JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
          JOIN pg_catalog.pg_locks         blocking_locks
               ON blocking_locks.locktype = blocked_locks.locktype
                   AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
                   AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
                   AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
                   AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
                   AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
                   AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
                   AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
                   AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
                   AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
                   AND blocking_locks.pid != blocked_locks.pid
          JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

## Autovacuum

### Autovacuum taking an access exclusive lock 

NOTE: [PostgreSQL VACUUM taking an access exclusive lock](https://blog.summercat.com/postgres-vacuum-taking-an-access-exclusive-lock.html) is becoming a problem on Aurora reader notes, as those will replicate the lock from the writer node.

```sql
-- disable truncate operation on autovacuum 
ALTER TABLE mytable SET (vacuum_truncate = off, toast.vacuum_truncate = off);
```

```sql
-- list tables with where options (e.g. autovacuum) are set
select relname, reloptions, pg_namespace.nspname
from pg_class join pg_namespace on pg_namespace.oid = pg_class.relnamespace
where pg_namespace.nspname = 'public' and reloptions is not null;
```

```sql
-- running VACUUM with TRUNCATE operation manually
VACUUM (TRUNCATE) mytable
```
