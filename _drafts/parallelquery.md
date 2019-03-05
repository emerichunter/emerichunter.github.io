## Documentation 



## Configuration

\! vim /etc/postgresql/10/main/postgresql.conf

# - Asynchronous Behavior -

#effective_io_concurrency = 1           # 1-1000; 0 disables prefetching
#max_worker_processes = 8               # (change requires restart)
max_parallel_workers_per_gather = 4     # taken from max_parallel_workers
max_parallel_workers = 4                # maximum number of max_worker_processes that
                                        # can be used in parallel queries
#old_snapshot_threshold = -1            # 1min-60d; -1 disables; 0 is immediate
                                        # (change requires restart)
#backend_flush_after = 0                # measured in pages, 0 disables



dynamic_shared_memory_type = posix      # the default is the first option


## Test avant redémarrage

pgbench=# \dt+ public.*
                          List of relations
 Schema |       Name       | Type  |  Owner   |  Size   | Description 
--------+------------------+-------+----------+---------+-------------
 public | pgbench_accounts | table | postgres | 128 MB  | 
 public | pgbench_branches | table | postgres | 40 kB   | 
 public | pgbench_history  | table | postgres | 0 bytes | 
 public | pgbench_tellers  | table | postgres | 40 kB   | 
(4 rows)

pgbench=# select count(*) FROM pgbench_accounts ;
  count  
---------
 1000000
(1 row)

pgbench=# EXPLAIN select count(*) FROM pgbench_accounts ;
                                                              QUERY PLAN                                                               
---------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=22188.97..22188.98 rows=1 width=8)
   ->  Gather  (cost=22188.76..22188.97 rows=2 width=8)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=21188.76..21188.77 rows=1 width=8)
               ->  Parallel Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..20147.09 rows=416667 width=0)
(5 rows)

## Redémarrage

pgbench=# \! vim /etc/postgresql/10/main/postgresql.conf
pgbench=# \! pg_ctlcluster 10 main restart
Error: cluster is running from systemd, can only restart it as root. Try instead:
  sudo systemctl restart postgresql@10-main
pgbench=# \!   sudo systemctl restart postgresql@10-main

# Parallel Index Only Scan
pgbench=# EXPLAIN select count(*) FROM pgbench_accounts ;
                                                              QUERY PLAN                                                               
---------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=20105.84..20105.85 rows=1 width=8)
   ->  Gather  (cost=20105.42..20105.83 rows=4 width=8)
         Workers Planned: 4
         ->  Partial Aggregate  (cost=19105.42..19105.43 rows=1 width=8)
               ->  Parallel Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..18480.42 rows=250000 width=0)
(5 rows)

# Parallel Seq Scan 
pgbench=# EXPLAIN select * FROM pgbench_accounts where bid = 1000 ;
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Gather  (cost=1000.00..21426.36 rows=1 width=97)
   Workers Planned: 3
   ->  Parallel Seq Scan on pgbench_accounts  (cost=0.00..20426.26 rows=1 width=97)
         Filter: (bid = 1000)
(4 rows)


## Paramètres - Comment hacker le parallel query 

pgbench=# \! psql -c 'show all' | grep parallel
 force_parallel_mode                 | off                                     | Forces use of parallel query facilities.
 max_parallel_workers                | 4                                       | Sets the maximum number of parallel workers than can be active at one time.
 max_parallel_workers_per_gather     | 4                                       | Sets the maximum number of parallel processes per executor node.
 min_parallel_index_scan_size        | 512kB                                   | Sets the minimum amount of index data for a parallel scan.
 min_parallel_table_scan_size        | 8MB                                     | Sets the minimum amount of table data for a parallel scan.
 parallel_setup_cost                 | 1000                                    | Sets the planner's estimate of the cost of starting up worker processes for parallel query.
 parallel_tuple_cost                 | 0.1                                     | Sets the planner's estimate of the cost of passing each tuple (row) from worker to master backend.

## Force brute : activation pour la session 
*pgbench=# EXPLAIN select aid FROM pgbench_accounts where aid =10000 ;
                                            QUERY PLAN                                             
---------------------------------------------------------------------------------------------------
 Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..4.44 rows=1 width=4)
   Index Cond: (aid = 10000)
(2 rows)

pgbench=# set force_parallel_mode ='on' ;
SET
pgbench=# EXPLAIN select aid FROM pgbench_accounts where aid =10000 ;
                                               QUERY PLAN                                                
---------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.42..1004.54 rows=1 width=4)
   Workers Planned: 1
   Single Copy: true
   ->  Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..4.44 rows=1 width=4)
         Index Cond: (aid = 10000)
(5 rows)

## EXPLAIN ANALYZE 
pgbench=# EXPLAIN ANALYZE select aid FROM pgbench_accounts where aid =10000 ;
                                                                    QUERY PLAN                                                                     
---------------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.42..1004.54 rows=1 width=4) (actual time=10.510..10.513 rows=1 loops=1)
   Workers Planned: 1
   Workers Launched: 1
   Single Copy: true
   ->  Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..4.44 rows=1 width=4) (actual time=0.204..0.207 rows=1 loops=1)
         Index Cond: (aid = 10000)
         Heap Fetches: 0
 Planning time: 0.215 ms
 Execution time: 15.917 ms
(9 rows)

pgbench=# set force_parallel_mode ='off' ;
SET
pgbench=# EXPLAIN ANALYZE select aid FROM pgbench_accounts where aid =10000 ;
                                                                 QUERY PLAN                                                                  
---------------------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..4.44 rows=1 width=4) (actual time=0.103..0.106 rows=1 loops=1)
   Index Cond: (aid = 10000)
   Heap Fetches: 0
 Planning time: 0.172 ms
 Execution time: 0.160 ms
(5 rows)

=> ça ne vaut pas le coup, mais il semblerait que le paramètre  `parallel_setup_cost` soit surévalué.

Ce sont des variables de sessions, ce qui signifie qu'elles reprennent leur valeur par défaut pour une nouvelle connexion. 
You are now connected to database "pgbench" as user "postgres".
pgbench=# show force_parallel_mode 
pgbench-# ;
 force_parallel_mode 
---------------------
 off
(1 row)

pgbench=# show parallel_setup_cost ;
 parallel_setup_cost 
---------------------
 1000
(1 row)

