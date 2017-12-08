---
published: false
---
## A New Post
+++
categories = ["technique","veille"]
title = "Hacker la parallélisation dans PostgreSQL 10"
author = "Emeric Tabakhoff"
banner = "banners/elephants-family-clone.jpg"
tags = ["administration","optimisation","performances"]
description = "Qu'est-ce que la parallélisation ? Comment la paramétrer ? Que peut-on faire avec ?"
date = "2017-11-30T11:54:47+01:00"
draft = true
shortstory = ""

+++


Avec PostgreSQL version 10 qui arrive, c'est également davantage de parallèlisme.
Naturellement, cela fait soulever plusieurs questions&nbsp;:

* Que fait le parallélisme ?
* Quelle type de requête/tâche couvre le parallélisme ?
* Comment l'utiliser ?
* Quel gain peut-on obtenir de l'utilisation de la parallélisation ?
* Quelles sont les limites ?

Pour débuter, voyons ensemble comment faire le paramétrage global, après avoir exposé les principes théoriques. 
Dans un second temps, nous allons voir comment paramétrer le requêtage en parallèle pour une utilisation ponctuelle.


## Un peu de théorie 

La parallélisation de requête permet de lancer plusieurs processus pour effectuer, par exemple, le scan entier d'une table, comme son nom l'indique, en parallèle.
En théorie, cela permet de diviser le temps d'exécution d'une tâche par le nombre de processus lancés.
En revanche, cela multiplie d'autant la charge sur la machine sur laquelle s'effectue l'opération.

Je vous donne le lien vers la documentation officielle [ici](https://www.postgresql.org/docs/current/static/how-parallel-query-works.html).

Voici ce que couvrait déjà la parallélisation dans la précédente version (9.6)&nbsp;:

* Jointure et agrégat (Parallel JOIN, aggregate)
* Requête (Parallel query)
* Scan de table entière (Parallel seqscan)

Ce qui est couvert par la nouvelle version (10)&nbsp;:

* Parallel bitmap heap scans 	
* Parallel B-tree index scans*	
* Parallel merge joins

*Sous entendu également `Index Only Scan`.

Voici la liste exhaustive des améliorations apportées, [ici](https://wiki.postgresql.org/wiki/New_in_postgres_10#Additional_Parallelism_in_Query_Execution)

## Configuration

Après une lecture de la documentation, on constate que l'activation de la parallélisation ne necessite que 2 paramètres&nbsp;:

* `max_parallel_workers_per_gather` postitionné à 2 ou plus (2 étant la valeur par défaut).
* `dynamic_shared_memory_type` positionné à une valeur quelconque (`posix` par défaut).

>**NOTE**&nbsp;: Toutes les commandes shell sont en réalité effectuées dans `psql` avec la commande `\!` qui nous autorise à tout faire dans psql sans se déconnecter/reconnecter de façon répétée&nbsp;: 

     psql -d pgbench
     \! vim /etc/postgresql/10/main/postgresql.conf
~~~
# - Asynchronous Behavior -

...
max_parallel_workers_per_gather = 4     # taken from max_parallel_workers
max_parallel_workers = 4                # maximum number of max_worker_processes that
                                        # can be used in parallel queries
...

dynamic_shared_memory_type = posix      # the default is the first option
~~~

Voici donc les seules lignes qu'il est nécessaire de modifier pour activer ou plutôt ajuster la parallélisation (réglée à 2 processus par défaut).
Je place également le paramètre `max_parallel_workers` à 4 et je laisse le paramètre `max_worker_processes` à la valeur par défaut (8).

## Test avant rechargement du fichier de configuration

Lançons une requête dans la base `pgbench` créée au préalable.

~~~
createdb pgbench
pgbench -i -s 100 pgbench
~~~

~~~
pgbench=# \dt+ public.*
                          List of relations
 Schema |      Name       | Type  |  Owner   |  Size   | Description 
--------+------------------+-------+----------+---------+-------------
 public | pgbench_accounts | table | postgres | 1281 MB | 
 public | pgbench_branches | table | postgres | 40 kB   | 
 public | pgbench_history  | table | postgres | 0 bytes | 
 public | pgbench_tellers  | table | postgres | 80 kB   | 
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
~~~

Nous pouvons dores et déjà observer 2 _workers_ dans le plan d'exécution.

## Rechargement de la configuration

Tout simplement&nbsp;:

~~~
pgbench=# \! pg_ctlcluster 10 main reload
~~~


## Quelques exemples

### Parallel Index Only Scan

~~~
pgbench=# EXPLAIN select count(*) FROM pgbench_accounts ;
                                                              QUERY PLAN                                                               
---------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=20105.84..20105.85 rows=1 width=8)
   ->  Gather  (cost=20105.42..20105.83 rows=4 width=8)
         Workers Planned: 4
         ->  Partial Aggregate  (cost=19105.42..19105.43 rows=1 width=8)
               ->  Parallel Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..18480.42 rows=250000 width=0)
(5 rows)
~~~

### Parallel Seq Scan 

~~~
pgbench=# EXPLAIN select * FROM pgbench_accounts where bid = 1000 ;
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Gather  (cost=1000.00..21426.36 rows=1 width=97)
   Workers Planned: 3
   ->  Parallel Seq Scan on pgbench_accounts  (cost=0.00..20426.26 rows=1 width=97)
         Filter: (bid = 1000)
(4 rows)
~~~

Ici, on peut observer 3 _workers_&nbsp;: explication dans la partie [Limites](#limites).

### Parallel Index Scan

Sans parallelisation&nbsp;:
~~~
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select count(aid) from pgbench_accounts where aid > 100 and aid < 1000000 and bid > 1000 and bid < 900000;
                                                                         QUERY PLAN                                                                          
-------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=52637.36..52637.37 rows=1 width=8) (actual time=213.933..213.933 rows=1 loops=1)
   Output: count(aid)
   ->  Index Scan using pgbench_accounts_pkey on public.pgbench_accounts  (cost=0.43..52637.36 rows=1 width=4) (actual time=213.928..213.928 rows=0 loops=1)
         Output: aid, bid, abalance, filler
         Index Cond: ((pgbench_accounts.aid > 100) AND (pgbench_accounts.aid < 1000000))
         Filter: ((pgbench_accounts.bid > 1000) AND (pgbench_accounts.bid < 900000))
         Rows Removed by Filter: 999899
 Planning time: 0.452 ms
 Execution time: 214.017 ms
(9 rows)

~~~

Avec la parallélisation&nbsp;: 
~~~
pgbench=# set max_parallel_workers_per_gather =4                                                                                               
;
SET
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select count(aid) from pgbench_accounts where aid > 100 and aid < 1000000 and bid > 1000 and bid < 900000;
                                                                                 QUERY PLAN                                                                                 
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=41339.80..41339.81 rows=1 width=8) (actual time=136.008..136.009 rows=1 loops=1)
   Output: count(aid)
   ->  Gather  (cost=10.44..41339.80 rows=1 width=4) (actual time=136.004..136.004 rows=0 loops=1)
         Output: aid
         Workers Planned: 4
         Workers Launched: 4
         ->  Parallel Index Scan using pgbench_accounts_pkey on public.pgbench_accounts  (cost=0.43..41329.79 rows=1 width=4) (actual time=118.891..118.891 rows=0 loops=5)
               Output: aid
               Index Cond: ((pgbench_accounts.aid > 100) AND (pgbench_accounts.aid < 1000000))
               Filter: ((pgbench_accounts.bid > 1000) AND (pgbench_accounts.bid < 900000))
               Rows Removed by Filter: 199980
               Worker 0: actual time=102.728..102.728 rows=0 loops=1
               Worker 1: actual time=121.492..121.492 rows=0 loops=1
               Worker 2: actual time=118.803..118.803 rows=0 loops=1
               Worker 3: actual time=118.821..118.821 rows=0 loops=1
 Planning time: 0.505 ms
 Execution time: 137.774 ms
(17 rows)

~~~

## Paramètrage - Comment hacker le parallel query 

Voici la liste des paramètres de sessions sur lesquels nous pouvons jouer pour déclencher ou augmenter la parallélisation.

~~~
pgbench=# \! psql -c 'show all' | grep parallel
 force_parallel_mode                 | off                                     | Forces use of parallel query facilities.
 max_parallel_workers                | 4                                       | Sets the maximum number of parallel workers than can be active at one time.
 max_parallel_workers_per_gather     | 4                                       | Sets the maximum number of parallel processes per executor node.
 min_parallel_index_scan_size        | 512kB                                   | Sets the minimum amount of index data for a parallel scan.
 min_parallel_table_scan_size        | 8MB                                     | Sets the minimum amount of table data for a parallel scan.
 parallel_setup_cost                 | 1000                                    | Sets the planner's estimate of the cost of starting up worker processes for parallel query.
 parallel_tuple_cost                 | 0.1                                     | Sets the planner's estimate of the cost of passing each tuple (row) from worker to master backend.
~~~

* `force_parallel_mode`&nbsp;: Force l'utilisation des requêtes parllélisées.
* **`max_parallel_workers`**&nbsp;: _C'est un nouveau paramètre en version 10_. Configure le nombre maximum de _workers_ parallèles actifs au même instant.
* `max_parallel_workers_per_gather`&nbsp;: Configure le nombre maximum de processus parallèles par nœud executeur.
* **`min_parallel_index_scan_size`**&nbsp;: Remplace `min_parallel_relation_size`. Configure la taille minimum des données de l'index pour un scan parallèle.
* **`min_parallel_table_scan_size`**&nbsp;: Configure la taille minimum des données de la table pour un scan parallèle.
* `parallel_setup_cost`&nbsp;: Configure le coût du démarrage des processus _workers_ des requêtes parallèles pour le planner.
* `parallel_tuple_cost` &nbsp;: Configure le coût du passage de chaque tuple du _worker_ au programme principal.

## Force brute&nbsp;: activation pour la session 

Le requêtage en parallèle se déclenche automatiquement lorsque l'optimiseur trouve que les performances de la requête peuvent bénéficier de son activation.
Il arrive qu'il se trompe, il arrive que les statistiques ne soient pas encore à jour, dans ces cas là, une activation forcée peut être une solution.

Plan d'exécution avant activation&nbsp;:

~~~
*pgbench=# EXPLAIN select aid FROM pgbench_accounts where aid =10000 ;
                                            QUERY PLAN                                             
---------------------------------------------------------------------------------------------------
 Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..4.44 rows=1 width=4)
   Index Cond: (aid = 10000)
(2 rows)
~~~

Avec l'activation du mode `parallel`&nbsp;:

~~~
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
~~~

Voyez comme le plan d'exécution est changé dans ses premières lignes.
Il est également possible d'augmenter ou diminuer (voire désactiver) la parallélisation en modifiant le paramètre `max_parallel_workers_per_gather` à la volée.


## EXPLAIN ANALYZE VERBOSE pour obtenir le détail de chaque worker


Sans forçage du mode parallel query&nbsp;:
~~~
pgbench=# set force_parallel_mode ='off' ;
SET
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select aid FROM pgbench_accounts where aid =10000 ;
                                                                     QUERY PLAN                                                                     
----------------------------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using pgbench_accounts_pkey on public.pgbench_accounts  (cost=0.43..4.45 rows=1 width=4) (actual time=0.054..0.058 rows=1 loops=1)
   Output: aid
   Index Cond: (pgbench_accounts.aid = 10000)
   Heap Fetches: 0
 Planning time: 0.223 ms
 Execution time: 0.177 ms
(6 rows)

~~~

Avec l'activation&nbsp;:

~~~
pgbench=# set force_parallel_mode =on ;
SET
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select aid FROM pgbench_accounts where aid =10000 ;
                                                                        QUERY PLAN                                                                        
----------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.43..1004.55 rows=1 width=4) (actual time=1.894..1.895 rows=1 loops=1)
   Output: aid
   Workers Planned: 1
   Workers Launched: 1
   Single Copy: true
   ->  Index Only Scan using pgbench_accounts_pkey on public.pgbench_accounts  (cost=0.43..4.45 rows=1 width=4) (actual time=0.012..0.013 rows=1 loops=1)
         Output: aid
         Index Cond: (pgbench_accounts.aid = 10000)
         Heap Fetches: 0
         Worker 0: actual time=0.012..0.013 rows=1 loops=1
 Planning time: 0.044 ms
 Execution time: 2.995 ms
(12 rows)
~~~

Ici, ça n'en vaut pas la peine (voir le temps plus élevé avec la parallèlisation), mais il semblerait que le paramètre `parallel_setup_cost` soit surévalué (par défaut 1000).

### Agrégat


Sans parallélisation&nbsp;:

~~~
pgbench=# set max_parallel_workers_per_gather =0
;
SET
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select count(*) FROM pgbench_accounts a JOIN pgbench_tellers t ON a.bid= t.bid  ;
                                                                     QUERY PLAN                                                                     
----------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=1663963.48..1663963.49 rows=1 width=8) (actual time=15186.597..15186.597 rows=1 loops=1)
   Output: count(*)
   ->  Hash Join  (cost=28.50..1413963.48 rows=99999998 width=0) (actual time=1.822..9736.886 rows=100000000 loops=1)
         Hash Cond: (a.bid = t.bid)
         ->  Seq Scan on public.pgbench_accounts a  (cost=0.00..263935.00 rows=10000000 width=4) (actual time=0.090..971.920 rows=10000000 loops=1)
               Output: a.aid, a.bid, a.abalance, a.filler
         ->  Hash  (cost=16.00..16.00 rows=1000 width=4) (actual time=1.699..1.699 rows=1000 loops=1)
               Output: t.bid
               Buckets: 1024  Batches: 1  Memory Usage: 44kB
               ->  Seq Scan on public.pgbench_tellers t  (cost=0.00..16.00 rows=1000 width=4) (actual time=0.074..0.795 rows=1000 loops=1)
                     Output: t.bid
 Planning time: 0.614 ms
 Execution time: 15186.707 ms
(13 rows)
~~~

Avec parallélisation&nbsp;:

~~~
pgbench=# set max_parallel_workers_per_gather =4
;
SET
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select count(*) FROM pgbench_accounts a JOIN pgbench_tellers t ON a.bid= t.bid  ;
                                                                              QUERY PLAN                                                                               
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=539063.55..539063.56 rows=1 width=8) (actual time=8150.517..8150.517 rows=1 loops=1)
   Output: count(*)
   ->  Gather  (cost=539063.49..539063.54 rows=4 width=8) (actual time=8150.501..8150.510 rows=5 loops=1)
         Output: (PARTIAL count(*))
         Workers Planned: 4
         Workers Launched: 4
         ->  Partial Aggregate  (cost=538963.49..538963.50 rows=1 width=8) (actual time=8136.814..8136.814 rows=1 loops=5)
               Output: PARTIAL count(*)
               Worker 0: actual time=8133.955..8133.955 rows=1 loops=1
               Worker 1: actual time=8135.044..8135.045 rows=1 loops=1
               Worker 2: actual time=8127.795..8127.796 rows=1 loops=1
               Worker 3: actual time=8137.604..8137.604 rows=1 loops=1
               ->  Hash Join  (cost=28.50..476463.49 rows=25000000 width=0) (actual time=5.818..5172.453 rows=20000000 loops=5)
                     Hash Cond: (a.bid = t.bid)
                     Worker 0: actual time=3.222..5162.964 rows=19102760 loops=1
                     Worker 1: actual time=3.056..5225.835 rows=19798160 loops=1
                     Worker 2: actual time=3.383..5207.638 rows=21981960 loops=1
                     Worker 3: actual time=17.254..5126.673 rows=17479810 loops=1
                     ->  Parallel Seq Scan on public.pgbench_accounts a  (cost=0.00..188935.00 rows=2500000 width=4) (actual time=0.043..460.342 rows=2000000 loops=5)
                           Output: a.aid, a.bid, a.abalance, a.filler
                           Worker 0: actual time=0.025..489.930 rows=1910276 loops=1
                           Worker 1: actual time=0.027..448.600 rows=1979816 loops=1
                           Worker 2: actual time=0.119..456.790 rows=2198196 loops=1
                           Worker 3: actual time=0.027..454.902 rows=1747981 loops=1
                     ->  Hash  (cost=16.00..16.00 rows=1000 width=4) (actual time=2.103..2.103 rows=1000 loops=5)
                           Output: t.bid
                           Buckets: 1024  Batches: 1  Memory Usage: 44kB
                           Worker 0: actual time=2.398..2.398 rows=1000 loops=1
                           Worker 1: actual time=2.187..2.187 rows=1000 loops=1
                           Worker 2: actual time=2.490..2.490 rows=1000 loops=1
                           Worker 3: actual time=1.316..1.316 rows=1000 loops=1
                           ->  Seq Scan on public.pgbench_tellers t  (cost=0.00..16.00 rows=1000 width=4) (actual time=0.038..0.940 rows=1000 loops=5)
                                 Output: t.bid
                                 Worker 0: actual time=0.042..1.096 rows=1000 loops=1
                                 Worker 1: actual time=0.056..1.032 rows=1000 loops=1
                                 Worker 2: actual time=0.042..1.064 rows=1000 loops=1
                                 Worker 3: actual time=0.032..0.601 rows=1000 loops=1
 Planning time: 0.587 ms
 Execution time: 8154.206 ms
(39 rows)
~~~

Le temps d'exécution est seulement divisé par 2 malgré le nombre de `workers` porté à 4.
Voici le résultat pour la meme requête avec le paramétrage par défaut&nbsp;:

~~~
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select count(*) FROM pgbench_accounts a JOIN pgbench_tellers t ON a.bid= t.bid  ;
                                                                              QUERY PLAN                                                                               
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=789963.74..789963.75 rows=1 width=8) (actual time=8088.269..8088.270 rows=1 loops=1)
   Output: count(*)
   ->  Gather  (cost=789963.53..789963.74 rows=2 width=8) (actual time=8088.248..8088.263 rows=3 loops=1)
         Output: (PARTIAL count(*))
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=788963.53..788963.54 rows=1 width=8) (actual time=8079.017..8079.017 rows=1 loops=3)
               Output: PARTIAL count(*)
               Worker 0: actual time=8074.162..8074.162 rows=1 loops=1
               Worker 1: actual time=8075.583..8075.583 rows=1 loops=1
               ->  Hash Join  (cost=28.50..684796.86 rows=41666666 width=0) (actual time=2.636..5170.125 rows=33333333 loops=3)
                     Hash Cond: (a.bid = t.bid)
                     Worker 0: actual time=3.340..5135.878 rows=45583470 loops=1
                     Worker 1: actual time=2.933..5168.344 rows=28670260 loops=1
                     ->  Parallel Seq Scan on public.pgbench_accounts a  (cost=0.00..205601.67 rows=4166667 width=4) (actual time=0.081..467.744 rows=3333333 loops=3)
                           Output: a.aid, a.bid, a.abalance, a.filler
                           Worker 0: actual time=0.094..540.870 rows=4558347 loops=1
                           Worker 1: actual time=0.094..436.167 rows=2867026 loops=1
                     ->  Hash  (cost=16.00..16.00 rows=1000 width=4) (actual time=1.990..1.990 rows=1000 loops=3)
                           Output: t.bid
                           Buckets: 1024  Batches: 1  Memory Usage: 44kB
                           Worker 0: actual time=2.415..2.415 rows=1000 loops=1
                           Worker 1: actual time=2.059..2.059 rows=1000 loops=1
                           ->  Seq Scan on public.pgbench_tellers t  (cost=0.00..16.00 rows=1000 width=4) (actual time=0.055..0.930 rows=1000 loops=3)
                                 Output: t.bid
                                 Worker 0: actual time=0.061..1.093 rows=1000 loops=1
                                 Worker 1: actual time=0.064..0.959 rows=1000 loops=1
 Planning time: 2.373 ms
 Execution time: 8090.127 ms
(29 rows)
~~~

Le temps d'exécution est identique.
Le forçage ne semble pas donner de meilleure performance.
On peut raisonnablement penser que les statistiques sont à jour et font fonctionner la parallélisation au mieux.

### Parallel Merge Join 

~~~
pgbench=# set enable_nestloop =off ;
SET
pgbench=# set enable_hashjoin =off ;
SET
pgbench=# EXPLAIN (ANALYZE,  VERBOSE) select a2.bid, count(a1.aid) from pgbench_accounts a1 JOIN pgbench_accounts a2 ON a2.bid =a1.aid where a1.aid > 100 and a1.aid < 1000000 and a1.bid > 1000 and a1.bid < 900000 and a2.aid > 1000 and a2.aid <  5000000 GROUP BY a2.bid ;
                                                                                             QUERY PLAN                                                                                              
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=346416.58..405241.27 rows=1 width=12) (actual time=2063.483..2063.483 rows=0 loops=1)
   Output: a2.bid, count(a1.aid)
   Group Key: a2.bid
   ->  Gather Merge  (cost=346416.58..405241.26 rows=1 width=8) (actual time=2063.480..2063.480 rows=0 loops=1)
         Output: a2.bid, a1.aid
         Workers Planned: 4
         Workers Launched: 4
         ->  Merge Join  (cost=346406.52..405231.18 rows=1 width=8) (actual time=2013.947..2013.947 rows=0 loops=5)
               Output: a1.aid, a2.bid
               Inner Unique: true
               Merge Cond: (a2.bid = a1.aid)
               Worker 0: actual time=1999.291..1999.291 rows=0 loops=1
               Worker 1: actual time=2048.134..2048.134 rows=0 loops=1
               Worker 2: actual time=1996.137..1996.137 rows=0 loops=1
               Worker 3: actual time=1963.740..1963.740 rows=0 loops=1
               ->  Sort  (cost=346285.39..349439.60 rows=1261684 width=4) (actual time=1584.377..1584.377 rows=1 loops=5)
                     Output: a2.bid
                     Sort Key: a2.bid
                     Sort Method: external sort  Disk: 11904kB
                     Worker 0: actual time=1573.838..1573.838 rows=1 loops=1
                     Worker 1: actual time=1577.132..1577.132 rows=1 loops=1
                     Worker 2: actual time=1578.795..1578.795 rows=1 loops=1
                     Worker 3: actual time=1579.309..1579.309 rows=1 loops=1
                     ->  Parallel Index Scan using pgbench_accounts_pkey on public.pgbench_accounts a2  (cost=0.43..201181.65 rows=1261684 width=4) (actual time=0.169..599.276 rows=999800 loops=5)
                           Output: a2.bid
                           Index Cond: ((a2.aid > 1000) AND (a2.aid < 5000000))
                           Worker 0: actual time=0.216..625.253 rows=863394 loops=1
                           Worker 1: actual time=0.236..536.566 rows=1234591 loops=1
                           Worker 2: actual time=0.100..635.723 rows=892674 loops=1
                           Worker 3: actual time=0.202..585.567 rows=1139724 loops=1
               ->  Index Scan using pgbench_accounts_pkey on public.pgbench_accounts a1  (cost=0.43..52637.36 rows=1 width=4) (actual time=429.567..429.567 rows=0 loops=5)
                     Output: a1.aid, a1.bid, a1.abalance, a1.filler
                     Index Cond: ((a1.aid > 100) AND (a1.aid < 1000000))
                     Filter: ((a1.bid > 1000) AND (a1.bid < 900000))
                     Rows Removed by Filter: 999899
                     Worker 0: actual time=425.448..425.448 rows=0 loops=1
                     Worker 1: actual time=470.998..470.998 rows=0 loops=1
                     Worker 2: actual time=417.339..417.339 rows=0 loops=1
                     Worker 3: actual time=384.427..384.427 rows=0 loops=1
 Planning time: 1.098 ms
 Execution time: 2071.749 ms
(41 rows)
~~~

> **NOTE**&nbsp;: J'ai également forcé l'utilisation de `Merge Join` (en désactivant `Hash join` et `Nested Loop`). 
Il est généralement préférable de laisser l'optimiseur choisir. Ceci est simplement une illustration.

### Costs

Sans postionner le forçage du mode parallèle, on peut aussi simplement jouer sur les coûts (`costs`) et avoir de bonnes surprises&nbsp;:

~~~
pgbench=# set parallel_setup_cost=100 ;
pgbench=# set parallel_tuple_cost=0.005 ;
SET
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select aid FROM pgbench_accounts where aid >10000 and aid \<1000000;
                                                                                    QUERY PLAN                                                                                    
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=100.44..28449.78 rows=995734 width=4) (actual time=0.994..310.197 rows=989999 loops=1)
   Output: aid
   Workers Planned: 4
   Workers Launched: 4
   ->  Parallel Index Only Scan using pgbench_accounts_pkey on public.pgbench_accounts  (cost=0.43..23371.11 rows=248934 width=4) (actual time=0.174..38.678 rows=198000 loops=5)
         Output: aid
         Index Cond: ((pgbench_accounts.aid > 10000) AND (pgbench_accounts.aid \< 1000000))
         Heap Fetches: 0
         Worker 0: actual time=0.248..49.426 rows=238266 loops=1
         Worker 1: actual time=0.087..42.455 rows=260679 loops=1
         Worker 2: actual time=0.219..53.188 rows=227286 loops=1
         Worker 3: actual time=0.211..44.733 rows=260226 loops=1
 Planning time: 0.516 ms
 Execution time: 357.773 ms
(14 rows)

pgbench=# set parallel_tuple_cost=0.01 ;
SET
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select aid FROM pgbench_accounts where aid >10000 and aid \<1000000;
                                                                                QUERY PLAN                                                                                
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=100.44..40896.46 rows=995734 width=4) (actual time=6.191..443.597 rows=989999 loops=1)
   Output: aid
   Workers Planned: 1
   Workers Launched: 1
   Single Copy: true
   ->  Index Only Scan using pgbench_accounts_pkey on public.pgbench_accounts  (cost=0.43..30839.12 rows=995734 width=4) (actual time=0.077..135.431 rows=989999 loops=1)
         Output: aid
         Index Cond: ((pgbench_accounts.aid > 10000) AND (pgbench_accounts.aid \< 1000000))
         Heap Fetches: 0
         Worker 0: actual time=0.077..135.431 rows=989999 loops=1
 Planning time: 0.193 ms
 Execution time: 492.649 ms
(12 rows)
~~~

## LIMITES 

Le nombre de processus instanciés est conditionné par un cœfficient fixe (progression géométrique) non publié dans la documentation mais mentionné dans le code de PostgreSQL et dans certains articles anglophones comme [ici](https://blog.2ndquadrant.com/postgresql96-parallel-sequential-scan/).
Ce nombre est fonction de la taille minimale de l'objet considéré (index ou table) avec les paramètres `min_parallel_index_scan_size` et `min_parallel_table_scan_size`. 
Prenons une table de moins de 8Mo (paramètre par défaut), en dessous de cette limite 0 worker de déclenché. 
Entre 8 et 24 (3x8), 1 worker, entre 24 et 72 (3x24), 2 _workers_ et ainsi de suite...
C'est pour celà que vous n'aurez pas nécessairement le nombre maximum de processus déclenché à chaque requête car celui-ci dépend de la taille de la relation.
Vous pouvez très bien jouer sur ces limites pour déclencher plus de worker (diminuer la taille minimale) et consommer plus de ressources ou au contraire augmenter la limite pour en déclencher moins et 
ainsi consommer moins de ressources ([retour aux exemples](#parallel-seq-scan)).


**Important**&nbsp;: Le nombre de processus total de l'instance (parallèle et non-parallèle) est limité par `max_worker_processes`, celui-ci n'est modifiable qu'avec redémarrage de l'instance.
Il sert donc dans ce cas de garde fou, tant que celui-ci est raisonnablement positionné évidemment.
Lorsque l'on joue avec les paramètres pour augmenter la parallélisation, il faut tenir compte de cette limite dure sans quoi, dans le cas de connexions concurrentes importantes, 
l'accès à la parallélisation sera limité.

## Nettoyer apres son passage ?

Ce sont des variables de sessions (à l'exception de `max_worker_processes`), ce qui signifie qu'elles reprennent leur valeur par défaut pour une nouvelle connexion à l'instance 
et garde ces valeurs tout au long de la connexion. 
Si rien n'est mis dans le fichier de configuration, il n'y a rien à remettre à zéro.
Il est donc possible de faire du cas par cas en appliquant des valeurs différentes pour une requête gourmande lors d'une plage horaire avec un trafic moindre.
`SET` est par défaut valable pour la `session`. 
On peut utiliser `SET LOCAL` pour limiter ce paramétrage à la transaction en cours.

## Quelques conseils et mises en garde

Pour un bon **réglage de départ de l'instance**, je vous conseille de partir sur 
[`max_parallel_workers_per_gather`](https://www.postgresql.org/docs/current/static/runtime-config-resource.html#GUC-MAX-PARALLEL-WORKERS-PER-GATHER) 
égalant le nombre de cœurs (virtuels ou non).
Vous pouvez regler [`max_parallel_workers`](https://www.postgresql.org/docs/current/static/runtime-config-resource.html#GUC-MAX-PARALLEL-WORKERS) avec une valeur égale ou inférieure à 
[`max_worker_processes`](https://www.postgresql.org/docs/current/static/runtime-config-resource.html#GUC-MAX-WORKER-PROCESSES) dans le fichier de configuration `postgresql.conf`.

Quelques principes élémentaires&nbsp;:

* Pour **forcer la parallélisation**, `force_parallel_mode` à `ON`.

* Pour **déclencher la parallélisation** d'une requête, il peut aussi être indiqué de dimininuer `parallel_tuple_cost` (même en dessous de `cpu_tuple_cost`) et `parallel_setup_cost`=100 (par exemple) si vous constatez, comme moi, que celui-ci semble trop élevé..

* Pour **désactiver totalement la paralléllisation**, il suffit de placer `max_parallel_workers_per_gather =0` si le besoin s'en fait sentir.

* Ne modifier les autres paramètres que lors d'une **session** (ou **transaction**) pour ne pas parasiter le comportement de toutes les requêtes de votre instance.

* **Expérimentez** dans un **environnement de test** ousur un **réplicat**.

Pour finir, attention à un trop grand degré de parallélisation qui peut consommer d'avantage de mémoire et de CPU qu'attendu, ce qui pourrait engendrer des résultats contre-productifs 
(voir le résultat pour les agrégats).

Ces optimisations ont leur limites.
On ne peut pas imposer à proprement parler à l'optimiseur de faire une requête en parallèle simplement avec `force_parallel_mode`, 
cependant on peut tout à fait guider celui-ci vers ce choix de façon assez forte parfois avec de bonnes surprises et parfois avec de mauvaises.

A vos marques, prêts... Parallélisez&nbsp;!
