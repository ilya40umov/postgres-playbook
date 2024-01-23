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

```
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

### Get the total size of all indecies

```
-- get total size of indices
SELECT
    pg_size_pretty(sum(pg_relation_size(indexrelid))) as total_size_of_all_indices
FROM pg_stat_all_indexes JOIN pg_class ON pg_stat_all_indexes.relid = pg_class.oid
WHERE pg_stat_all_indexes.schemaname = 'public';
```

## Indices

### Get invalid indices

NOTE: You may end up with an "invalid" index if you try to create an index with unique constraint in background and this operation fails.

```
-- get invalid indices
SELECT
    *
FROM pg_class inner join pg_index ON pg_index.indexrelid = pg_class.oid
WHERE pg_index.indisvalid = false;
```

* [Problems with concurrent Postgres indexes](https://medium.com/carwow-product-engineering/problems-with-concurrent-postgres-indexes-and-how-to-solve-them-c57f7656c852)
