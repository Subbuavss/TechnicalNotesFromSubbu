### Problem: How to query the changes in WAL segments using LSN using temporary replication slot.

Note: remove the tempoary replication slot  (test_slot_1) once your research is done.  

```
-- 
-- Preconditions:
-- Created Publication and Subscription for table t2.
-- I can see data from table `t2` is replicated from master host to slave host
-- 
-- postgresql version master/slave
                      ^
db1=# select version();
                                                       version
---------------------------------------------------------------------------------------------------------------------
 PostgreSQL 17.2 (Debian 17.2-1.pgdg120+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
(1 row)


-- 
-- table `t2` 
-- 
db1=# select * from t2;
 id
----
  1
  2
  3
  4
  5
(5 rows)

-- 
-- Create tempoarary logilcal slot
-- 
db1=# SELECT * FROM pg_logical_slot_get_changes('test_slot_1', NULL, NULL, 'include-xids', '0');
-- 
-- 
db1=# select * from pg_replication_slots;

  slot_name  |    plugin     | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase |        inactive_since         | conflicting | invalidation_reason | failover | synced
-------------+---------------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------+-------------------------------+-------------+---------------------+----------+--------
 mysub_t1    | pgoutput      | logical   |  16386 | db1      | f         | t      |        169 |      |          765 | 0/1952690   | 0/19526C8           | reserved   |               | f         |                               | f           |                     | f        | f
 test_slot_1 | test_decoding | logical   |  16386 | db1      | f         | f      |            |      |          765 | 0/1952448   | 0/19526C8           | reserved   |               | f         | 2025-06-08 18:53:36.645569+00 | f           |                     | f        | f
(2 rows)

(END)
-- 
-- I can see `mysub_t1` is the replication slot that ensure the primary server retains Write-Ahead Log (WAL) files necessary for replicas
-- I can see `test_slot_1`  is the replication slot for research purpose to read the changes 
-- 
-- Insert a row to table t2
db1=#  insert into t2 values(6);
INSERT 0 1
-- 
-- 
-- query  pg_replication_slots;
-- 
db1=# select * from pg_replication_slots;

  slot_name  |    plugin     | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase |        inactive_since         | conflicting | invalidation_reason | failover | synced
-------------+---------------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------+-------------------------------+-------------+---------------------+----------+--------
 mysub_t1    | pgoutput      | logical   |  16386 | db1      | f         | t      |        169 |      |          766 | 0/1952840   | 0/1952878           | reserved   |               | f         |                               | f           |                     | f        | f
 test_slot_1 | test_decoding | logical   |  16386 | db1      | f         | f      |            |      |          765 | 0/1952448   | 0/19526C8           | reserved   |               | f         | 2025-06-08 18:53:36.645569+00 | f           |                     | f        | f
(2 rows)

(END)

-- 
-- 
-- Query  LSN `0/1952878` 
-- 
db1=# SELECT * FROM pg_logical_slot_get_changes('test_slot_1', '0/1952878', NULL);
    lsn    | xid |                  data
-----------+-----+----------------------------------------
 0/19526C8 | 765 | BEGIN 765
 0/19526C8 | 765 | table public.t2: INSERT: id[integer]:6
 0/1952738 | 765 | COMMIT 765
(3 rows)

db1=#
-- 
-- data coliumn contains  table name, insert command, column , insert value
-- 

-- 
--  Query pg_replication_slots and I can see confirmed_flush_lsn are having same LSN value. ie.. WAL segment is matured and removed
-- 
db1=# select * from pg_replication_slots;


  slot_name  |    plugin     | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase |        inactive_since         | conflicting | invalidation_reason | failover | synced
-------------+---------------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------+-------------------------------+-------------+---------------------+----------+--------
 mysub_t1    | pgoutput      | logical   |  16386 | db1      | f         | t      |        169 |      |          766 | 0/1952840   | 0/1952878           | reserved   |               | f         |                               | f           |                     | f        | f
 test_slot_1 | test_decoding | logical   |  16386 | db1      | f         | f      |            |      |          766 | 0/1952738   | 0/1952878           | reserved   |               | f         | 2025-06-08 19:04:57.149405+00 | f           |                     | f        | f
(2 rows)

-- 
-- From below comand, i can see WAL command is matured and removed
-- 
db1=# SELECT * FROM pg_logical_slot_get_changes('test_slot_1', '0/1952878', NULL);
 lsn | xid | data
-----+-----+------
(0 rows
```
