---
title: Cleaning PostgreSQL tables
date: 2010-05-26T17:52:00
summary: "Quick note on how making some cleanup in postgresql database"
tags:
  - postgresql
---
When a PostgreSQL database takes too much place, it may be time to make some cleanup. And sometimes, a big DELETE with a VACUUM FULL may not be enough. For example, today I've find out a few things to do to cleanup things more deeply.

I've done a simple "DELETE * FROM obj;", followed by a [VACUUM](http://www.postgresql.org/docs/current/static/sql-vacuum.html). I got back 300 MB, but my table is still using space:

```sql
engineProd=# SELECT count(*) FROM obj;
 count
-------
     0
(1 row)

engineProd=# SELECT pg_size_pretty(pg_total_relation_size('obj'));
 pg_size_pretty
----------------
 85 MB
(1 row)
```

It's possible with PostgreSQL to monitor easily disk storage used by tables and their indexes. You'll have to use database's internal tables, as pg_class:

{% highlight sql %}
engineProd=# SELECT relname, relpages FROM pg_class
EngineProd-# WHERE relpages > 1000 ORDER BY relpages DESC;
           relname           | relpages
-----------------------------+----------
 obj_hash_index              |     4322
 obj_action_created_at_index |     1586
 obj_created_at_index        |     1527
 obj_st_to_index             |     1262
 obj_st_from_index           |     1248
 obj_status_index            |     1224
 obj_pkey                    |     1223
```

The tricky part is to reindex with [REINDEX](http://www.postgresql.org/docs/current/static/sql-reindex.html):

```sql
engineProd=# REINDEX TABLE obj;
REINDEX
```

And the result is a lot better:

```sql
engineProd=# SELECT pg_size_pretty(pg_total_relation_size('obj'));
 pg_size_pretty 
----------------
 88 kB
(1 row)
```

Note that there is a full technical page about this in the [PostgreSQL manual](http://www.postgresql.org/docs/current/interactive/disk-usage.html).mycroft@nas0:/volume1/backup/dev/gazette/_posts$ 