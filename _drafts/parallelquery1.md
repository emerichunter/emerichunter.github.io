
layout: post
Title: Parallel query in PostgreSQL (part 1)
Draft: true
published: true

<!-- 2017-12-22   1995 mots, 10 minutes de lecture   Emeric Tabakhoff -->

<!-- Avec PostgreSQL version 10 qui arrive, c’est également davantage de parallélisme. Naturellement, cela fait soulever plusieurs questions : -->


PostgreSQL version 11 is now here. It's now the second improvement on parallism (introduction was made in 9.6). Of course, this raises several questions:

<!--     Que fait le parallélisme ?
    Quelle type de requête/tâche couvre le parallélisme ?
    Comment l’utiliser ?
    Quel gain peut-on obtenir ?
    Quelles sont les limites ? -->

  Un peu de théorie
    * What does parallel query do?
    * Which queries can be made parallel?
    * How can one use it?
    * What is there to gain?
    * What are the limits?

First, a little theory. Second, taking into account hardware we will adjust some parameters to get the best out of this feature.

<!--
Avant tout, un peu de théorie, en voyant ce que l’on peut attendre de la parallélisation dans cette nouvelle version. Ensuite, après avoir pris en considération les capacités de la machine, un ajustement des paramètres pour mieux tirer partie de cette fonctionnalité. -->

# A little theory

<!--  
Une requête SQL suit un plan d’exécution qui contient plusieurs étapes. Certaines d’entre elles sont amenées à traiter une quantité de données importantes. Il peut donc être opportun de diviser cette quantité de données en parts égales afin de la faire traiter par plusieurs processus, en parallèle. C’est ce que fait PostgreSQL avec les requêtes parallélisées. -->
A SQL query follows an execution plan which may contain several steps. Some of them might have to process a lot of data. In light of this, it might be a good idea to divide this amount of data in order to have multiple processes work through i in parallel. This is parallel queries in PostgreSQL.

<!--En théorie, cela permet de diviser le temps d’exécution d’une étape par le nombre de processus lancés.-->
In theory, this would allow to cut execution time by the number of process fired. But the cost of such an operation is to multiply load on the host (memory and CPU).

<!--En revanche, cela multiplie d’autant la charge (CPU et RAM) sur la machine sur laquelle s’effectue l’opération.-->


<!-- Voici ce que couvrait déjà le parallélisme dans la version 9.6 :-->
This is what parallel queries cover in version 9.6:

    * Parallel query
    * Parallel seqscan of and entire table
    * Parallel JOIN, aggregate)


<!-- Ce qui est couvert en plus dans la version 10 : -->
These are the features shipped with version 10:

    * Parallel bitmap heap scans
    * Parallel B-tree index scans (including Index Only Scan)
    * Parallel merge joins

Configuration par défaut

Selon la documentation, l’activation du parallélisme ne nécessite que 2 paramètres :

    max_parallel_workers_per_gather postitionné à 2 ou plus (2 étant la valeur par défaut)
    dynamic_shared_memory_type positionné à une valeur différente de none (posix par défaut)

La parallélisation est donc réglée à 2 processus par défaut.
Test “out of the box”

Initialisation d’une base avec pgbench, pour observer le parallélisme

createdb pgbench
pgbench -i -s 100 pgbench

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

Testons une requête dans la base que nous venons de créer.

pgbench=# EXPLAIN select count(*) FROM pgbench_accounts ;
                                                              QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=22188.97..22188.98 rows=1 width=8)
   ->  Gather  (cost=22188.76..22188.97 rows=2 width=8)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=21188.76..21188.77 rows=1 width=8)
               ->  Parallel Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..20147.09 rows=416667 width=0)
(5 rows)

Nous pouvons observer 2 workers planifiés dans le plan d’exécution, sans rien modifier. Un worker est une unité d’exécution parallèle pour une étape du plan de requêtage. Nous verrons plus loin que s’il est planifié, il ne sera pas nécessairement lancé.
Premier ajustement

Maintenant, adaptons la configuration à notre machine de test : c’est à dire pour 4 CPU, 4 processus parallèles.

Voici la modification du postgresql.conf

max_parallel_workers_per_gather = 4     # taken from max_parallel_workers
max_parallel_workers = 4                # maximum number of max_worker_processes that
                                        # can be used in parallel queries

    max_parallel_workers définit le nombre total de worker pour les requêtes parallèles parmi le nombre maximum de worker de l’instance définit par max_worker_processes.
    max_parallel_workers_per_gather définit le nombre maximum de workers par étape parallélisable du plan.

Rechargement de la configuration

Pour prendre en compte les modifications apportées à la configuration, on doit la recharger :

pgbench=# SELECT pg_reload_conf();

Quelques exemples
Parallel Index Only Scan

pgbench=# EXPLAIN select count(*) FROM pgbench_accounts ;
                                                              QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=20105.84..20105.85 rows=1 width=8)
   ->  Gather  (cost=20105.42..20105.83 rows=4 width=8)
         Workers Planned: 4
         ->  Partial Aggregate  (cost=19105.42..19105.43 rows=1 width=8)
               ->  Parallel Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..18480.42 rows=250000 width=0)
(5 rows)

Parallel Seq Scan

pgbench=# EXPLAIN select * FROM pgbench_accounts where bid = 1000 ;
                                     QUERY PLAN
------------------------------------------------------------------------------------
 Gather  (cost=1000.00..21426.36 rows=1 width=97)
   Workers Planned: 3
   ->  Parallel Seq Scan on pgbench_accounts  (cost=0.00..20426.26 rows=1 width=97)
         Filter: (bid = 1000)
(4 rows)

Ici, on peut observer seulement 3 workers planifiés : nous y reviendrons dans la partie Limites.
Parallel Index Scan

Avec la parallélisation :

pgbench=# EXPLAIN select count(aid) from pgbench_accounts where aid > 100 and aid < 1000000 and bid > 1000 and bid < 900000;
                                                      QUERY PLAN                                                       
-----------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=42329.90..42329.91 rows=1 width=8)
   ->  Gather  (cost=1000.43..42329.89 rows=1 width=4)
         Workers Planned: 4
         ->  Parallel Index Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.43..41329.79 rows=1 width=4)
               Index Cond: ((aid > 100) AND (aid < 1000000))
               Filter: ((bid > 1000) AND (bid < 900000))
(6 rows)


EXPLAIN ANALYZE VERBOSE pour obtenir le détail de chaque worker

Nous avons parlé plus haut du fait que les workers étaient planifiés, mais est-ce que dans les faits, cette planification est bien suivie ? Pour le savoir, il suffit d’utiliser les options (ANALYZE, VERBOSE) de la commande EXPLAIN.

pgbench=# EXPLAIN (ANALYZE, VERBOSE) select aid,bid FROM pgbench_accounts where aid > 10000 and bid> 1000 and bid \<100000;
                                                                QUERY PLAN                                                                 
-------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..208685.10 rows=1 width=8) (actual time=2650.805..2650.805 rows=0 loops=1)
   Output: aid, bid
   Workers Planned: 4
   Workers Launched: 4
   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..207685.00 rows=1 width=8) (actual time=2627.557..2627.557 rows=0 loops=5)
         Output: aid, bid
         Filter: ((pgbench_accounts.aid > 10000) AND (pgbench_accounts.bid > 1000) AND (pgbench_accounts.bid \< 100000))
         Rows Removed by Filter: 2000000
         Worker 0: actual time=2624.376..2624.376 rows=0 loops=1
         Worker 1: actual time=2624.425..2624.425 rows=0 loops=1
         Worker 2: actual time=2621.906..2621.906 rows=0 loops=1
         Worker 3: actual time=2621.377..2621.377 rows=0 loops=1
 Planning time: 2.391 ms
 Execution time: 2654.363 ms
(14 rows)

Agrégat

Sans parallélisation :

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

Avec parallélisation :

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

Le temps d’exécution est seulement divisé par 2 malgré le nombre de workers porté à 4. Voici le résultat pour la meme requête avec le paramétrage par défaut :

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

Le temps d’exécution est identique.
LIMITES

Le nombre de processus instanciés est conditionné par un cœfficient fixe (progression géométrique) non publié dans la documentation mais mentionné dans le code de PostgreSQL et dans certains articles anglophones comme ici.

commit : ici

fichiers :

    src/backend/optimizer/path/allpaths.c (lignes 2919-2955)
    src/backend/utils/misc/guc.c (ligne 2785)

Ce nombre est fonction de la taille minimale de l’objet considéré (index ou table) avec les paramètres min_parallel_index_scan_size et min_parallel_table_scan_size. Prenons une table de moins de 8Mo (paramètre par défaut), en dessous de cette limite aucun worker ne sera déclenché. Entre 8 et 24Mo (3x8), 1 worker, entre 24 et 72Mo (3x24), 2 workers et ainsi de suite… C’est pour celà que vous n’aurez pas nécessairement le nombre maximum de processus déclenchés à chaque requête car celui-ci dépend de la taille de la relation. Vous pouvez très bien jouer sur ces limites pour déclencher plus de worker (diminuer la taille minimale) et consommer plus de ressources ou au contraire augmenter la limite pour en déclencher moins et ainsi consommer moins de ressources (retour aux exemples). Il est bien sûr toujours préférable de tester à chaque changement de paramètre pour isoler la modification du comportement un à un pour chaque paramètre.

Important : Le nombre de processus total de l’instance (parallèle et non-parallèle) est limité par max_worker_processes, celui-ci n’est modifiable qu’avec redémarrage de l’instance. Il sert donc dans ce cas de garde fou, tant que celui-ci est raisonnablement positionné évidemment. Lorsque l’on joue avec les paramètres pour augmenter la parallélisation, il faut tenir compte de cette limite dure sans quoi, dans le cas de connexions concurrentes importantes, l’accès à la parallélisation sera limité.
Quelques conseils pour bien démarrer

Pour un bon réglage de départ de l’instance, je vous conseille de partir sur max_parallel_workers_per_gather égalant le nombre de cœurs (virtuels ou non). 1 worker par CPU. Vous pouvez régler max_parallel_workers_per_gather avec une valeur égale ou inférieure à max_worker_processes dans le fichier de configuration postgresql.conf.

Remarque : si l’architecture sur laquelle vous travaillez comporte de la réplication, c’est un processus supplémentaire pour l’instance à chaque réplica. Dans ce cas, je vous conseille de retirer autant de process autorisé par max_worker_processes que vous avez de réplica.

    Exemple : 8 CPUs - 2 réplicas = 6 max_worker_processes et entre 2 et 6 pour max_parallel_workers_per_gather.

Si vous souhaitez désactiver totalement ou limiter l’utilisation de cette fonctionnalité, il suffit de passer le premier paramètre à 0 (une valeur de 1 instancie un worker en lançant la parallélisation ce qui pour s’avérer couteux).

Rappel : ce paramètre se recharge avec reload sans redémarrer l’instance. Si c’est assez pour vous, c’est ici la fin de ce post. Si vous en voulez plus et que ce qui a été dit ici n’est pas suffisant pour vous, la suite au prochain numéro !

A vos marques, prêts… Parallélisez !
Notre flux
abonnez vous
Articles récents

    Les Procédures stockées dans PostgreSQL 11
    De retour de PGDay Paris
    De retour de Nordic PGDay
    PostgreSQL 10.3 et autres correctifs
    De retour du FOSDEM 2018

Catégories

    toutes (65)
    formation (5)
    loxodata (1)
    technique (29)
    veille (21)
    événement (20)
