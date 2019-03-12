+++
categories = ["technique","veille"]
title = "PostgreSQL et la parallélisation (partie 2)"
author = "Emeric Tabakhoff"
banner = "head/ganesha.jpg"
tags = ["administration","optimisation","performances"]
description = "PostgreSQL 10. Paramétrage de parallélisme de requête. Optimisation avancée adressée aux DBA."
date = "2017-12-22T11:54:47+01:00"
draft = true
shortstory = "PostgreSQL 10 améliore la parallélisation. Voyons ce qui change et ce qu'il est possible de faire avec. Les bonnes pratiques. Ce billet s'adresse aux experts."
+++

Dans la [précédente partie](/post/parallelquery1/), nous avons vu que PG10 couvrait une large palette d'utilisation et comment faire le paramétrage de prime abord.
Aujourd'hui, je vous propose de plonger plus profondément dans les rouages de ce mécanisme et de régler plus finement cette fonctionnalité.

Nous verrons ensemble les gains possibles, les limites et surtout comment hacker la parallélisation en forçant sont déclenchement de différentes manières.
Si la partie précédente couvrait le paramétrage général de l'instance, ce post va couvrir des besoins plus spécifiques et s'adresse plus particulièrement aux utilisateurs plus avancés.

<!-- JCA
Le mot expert me gène. Je préfére les «utilisateurs plus avancés». Les experts, c'est sensé être nous. :)
-->

Nous allons passer en revue les paramètres et voir comment les modifier brièvement mais pas seulement.


## Revenons sur notre configuration

<!-- JCA
je fusionnerais bien cette partie avec la suivante (paramétrage) en supprimant la partie sur le postgresql.conf

Tu peux débuter par, nous pouvons approfondir les paramètres de PostgreSQL concernant le parallélisme en requêtant pg_settings.
Nous y retrouverons la configuration que nous avions déjà adapté à notre machine, à savoir 4 _workers_ par étape du plan, choisi dans un _pool_ de 4 _workers, correspondant au nombre de CPU.

Note qu'on n'a jamais fait la corrélation CPU/Workers dans le premier article --ETA: corrigé dans pq1
-->


Nous pouvons approfondir les paramètres de PostgreSQL concernant le parallélisme en requêtant `pg_settings`.
Nous y retrouverons la configuration que nous avions déjà adapté à notre machine, à savoir 4 _workers_ par étape du plan, choisi dans un _pool_ de 4 _workers, correspondant au nombre de CPU.

<!-- 
~~~sql
max_parallel_workers_per_gather = 4     # taken from max_parallel_workers
max_parallel_workers = 4                # maximum number of max_worker_processes that
                                        # can be used in parallel queries
~~~


## Paramétrage 
-->


Voici la liste des paramètres sur lesquels nous pouvons jouer pour déclencher ou augmenter la parallélisation globale de l'instance ou lors d'une session ou même d'une transaction
~~~sql
pgbench=# select name, setting, short_desc from pg_settings where name ~ 'parallel';
 force_parallel_mode             | off     | Forces use of parallel query facilities.
 max_parallel_workers            | 4       | Sets the maximum number of parallel workers than can be active at one time.
 max_parallel_workers_per_gather | 4       | Sets the maximum number of parallel processes per executor node.
 min_parallel_index_scan_size    | 512kB   | Sets the minimum amount of index data for a parallel scan.
 min_parallel_table_scan_size    | 8MB     | Sets the minimum amount of table data for a parallel scan.
 parallel_setup_cost             | 1000    | Sets the planner's estimate of the cost of starting up worker processes for parallel query.
 parallel_tuple_cost             | 0.1     | Sets the planner's estimate of the cost of passing each tuple (row) from worker to master backend.
~~~

<!-- JCA
Pourquoi y a-t-il des clés de conf en gras et d'autres non ?
-->

Le précédent post couvrait déjà ceci (en **gras** les nouveaux paramètres)&nbsp;:

* **`max_parallel_workers`**&nbsp;: _C'est un nouveau paramètre en version 10_. Configure le nombre maximum de _workers_ parallèles actifs au même instant.
* `max_parallel_workers_per_gather`&nbsp;: Configure le nombre maximum de processus parallèles par nœud executeur.
<!-- JCA
noeud c'est bien mais dans la partie 1 j'ai parlé d'étapes, je ne 
sais pas comment on a parlé du planner et des nodes de l'arbre de 
query mais ça paraît plus «parlant» étape, si tu préfères noeud, 
il faudra probablement retrofitter également dans la partie 1 si tu as 
utilisé cet idiome --ETA: nœud exécuteur = worker d'apres ma compréhension
-->

Nous allons voir ce qu'il est possible de faire avec ces paramètres&nbsp;:

* `force_parallel_mode`&nbsp;: Force l'utilisation des requêtes parllélisées.
* `min_parallel_relation_size` est remplacé par 2 paramètres permettant de différentier index et table.
  * **`min_parallel_index_scan_size`**&nbsp;: Configure la taille minimum du bloc de données pour déclencher un _worker_ pour le scan d'index.
  * **`min_parallel_table_scan_size`**&nbsp;: Configure la taille minimum du bloc de données pour déclencher un _worker_ pour le scan de table.

<!-- JCA
Cela est-il intéressant de mentionner que ça remplace ? on n'en parle 
pas avant, on parle de PG10 et pas PG96 du coup on peut s'épargner cette info ? (sinon mettre uen footnote ?)

Configure la taille du «bloc de données»  pour déclencher 
un worker pour le scan d'index.
-->

<!-- JCA  
Configure la taille du «bloc de données»  pour déclencher 
un worker pour le scan de tables  
-->
* `parallel_setup_cost`&nbsp;: Configure le coût du démarrage des processus _workers_ des requêtes parallèles pour le planner.

<!-- JCA
Par worker ou pour démarrer l'ensemble des workers ?
-->

* `parallel_tuple_cost` &nbsp;: Configure le coût du passage de chaque tuple du _worker_ au programme principal.

## Force brute
<!--&nbsp;: activation pour la session-->

Le requêtage en parallèle se déclenche automatiquement lorsque l'optimiseur trouve que les performances de la requête peuvent bénéficier de son activation.
<!--Il arrive qu'il se trompe, il arrive que les statistiques ne soient pas encore à jour, dans ces cas là, une activation forcée peut être une solution.-->
On peut faire une activation forcée, mais cela n'enclenche pas nécessairement de nouveaux `workers`, voyez plutôt.


Plan d'exécution avant activation&nbsp;:

~~~sql
pgbench=# set force_parallel_mode ='off' ;
SET
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select aid FROM pgbench_accounts where aid = 10000 ;
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

~~~sql
pgbench=# set force_parallel_mode = on ;
SET
pgbench=# EXPLAIN (ANALYZE, VERBOSE) select aid FROM pgbench_accounts where aid = 10000 ;
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

Voyez comme le plan d'exécution est changé dans ses premières lignes.
Remarquez la ligne `Workers Planned: 1` ceci est un coût supplémentaire pour le planificateur.
Remarquez également la différence sur la première ligne avec le `cost` qui est proche de 1000 car même sans réelle parallélisation le moteur utilise un _worker_ ce qui a un coût.
Ici, ça n'en vaut pas la peine (voir le temps plus élevé avec la parallélisation), mais il semblerait que le paramètre `parallel_setup_cost` soit surévalué (par défaut 1000).

<!-- JCA
Je supprimerais bien cette partie agrégats pour ne plus mettre que le 
texte que je raccrocherais bien ici en disant : 
Dans le premier acticle de cette série, nous avions déjà identifié 
que le gain de la parallélisation pouvait être contreproductif 
(lien) ce qui vient étayer le propos. -- ETA: je ne comprends pas l'idée
-->

Je pense qu'on peut donc se dire que ce n'est pas ici la bonne façon de procéder.
Il est également possible d'augmenter ou diminuer (voire désactiver) la parallélisation en modifiant le paramètre `max_parallel_workers_per_gather` à la volée.

### Agrégat

Comme nous l'avons vu la fois précédente, le fait d'avoir plus de workers disponibles n'influe pas nécessairement sur la rapidité d'exécution ([voir cet exemple dans l'article précédent](/post/parallelquery1/#agrégat)).

### Parallel Merge Join

<!-- JCA
Je suis un peu sec là

Comme nous l'avons vu également dans le premier article de la série, 
la parallélisation des jointures par fusion (Merge joins) font également partie des 
améliorations apportées par PostgreSQL 10. Voici une 
illustration où j'ai volontairement interdit les _nested 
loops_ et les _hash joins_ afin de déclencher volontairement l'utilisation des merge 
joins. Bien sûr, l'optimiseur de planification de PostgreSQL est un 
outil qui produira toujours un meilleur plan, aussi il convient de le 
laisser faire :
-->


Comme nous l'avons vu également dans le premier article de la série, 
la parallélisation des jointures par fusion (Merge joins) fait également partie des 
améliorations apportées par PostgreSQL 10. Voici une 
illustration où j'ai interdit les _nested 
loops_ et les _hash joins_ afin de déclencher volontairement l'utilisation des merge 
joins. Bien sûr, l'optimiseur de planification de PostgreSQL est un 
outil qui produira toujours un meilleur plan, aussi il convient de le 
laisser faire :

~~~sql
pgbench=# set enable_nestloop = off ;
SET
pgbench=# set enable_hashjoin = off ;
SET
pgbench=# EXPLAIN (ANALYZE,  VERBOSE) select a2.bid, count(a1.aid) from pgbench_accounts a1 JOIN pgbench_accounts a2 ON a2.bid = a1.aid where a1.aid > 100 and a1.aid < 1000000 and a1.bid > 1000 and a1.bid < 900000 and a2.aid > 1000 and a2.aid <  5000000 GROUP BY a2.bid ;
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

<!-- JCA
Si tu utilises mon laïus d'au dessus, tu peux te débarrasser de cette 
note 
-->
<!--
> **NOTE**&nbsp;: J'ai forcé l'utilisation de `Merge Join` (en désactivant `Hash join` et `Nested Loop`).
Il est généralement préférable de laisser l'optimiseur choisir. Il s'agit simplement ici d'une illustration.
-->

### Jouer sur les coûts

<!-- JCA
### Jouer sur les coûts
-->

Il n'est pas nécessaire d'activer le forçage du mode parallèle (nous avons pu voir que ceci ne nous a rien apporté dans l'exemple [précédent](/post/parallelquery2/#force-brute)). 

<!-- JCA
WARNING WARNING #référence!!!! non renseigné --ETA:OK-->

On peut simplement jouer sur les coûts (`costs`) et avoir de bonnes surprises&nbsp;:

~~~sql
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

Ici, on gagne 150ms par rapport à la même requête sans parallélisation.
Cette optimisation, _a priori_ superflue apporte pourtant un gain en performance.

<!-- JCA

Je m'attends à une explication plus en profondeur ici, par exemple, 
«Le fait que le coût de passage des tuples soit deux fois supérieur 
implique que l'optimiseur choisira de ne lancer qu'un seul worker. 
On voit donc que l'ajustement de ce paramètre se doit de refléter la 
réalité de la configurtion machine (probablement le cache L1/L2) ? --ETA: je n'ai pas compris ce que tu entends par là

Quid des paramètres pour cette requête ?
  min_parallel_index_scan_size  min_parallel_table_scan_size --ETA: je n'y ai pas touché - valeur par défaut
  
  Non ?
-->

## LIMITES

<!-- JCA
limites n'existe pas/plus?
De toutes façons, on peut y faire appel plutôt en conclusion en 
disant qu'il existe des limites et qu'on peut les consulter <wherever>
-->

Si vous n'avez pas lu la première partie, vous pouvez lire le paragraphe sur les limites s'y rapportant [ici](/post/parallelquery1/#limites).

## Nettoyer apres son passage ?

Toutes ces variable sont des variables de sessions (à l'exception de 
`max_worker_processes`), ce qui signifie qu'elles reprennent, lors 
d'une nouvelle connexion, les valeurs qui leur sont attribuées dans le 
fichier de `postgresql.conf` ou leurs 
valeurs par défaut si elles n'y sont pas définies. Une fois définies 
en session, les nouvelles valeurs sont conservées jusqu'à déconnexion du 
client ou jusqu'au moment où elle seront redéfinies.

<!-- JCA
Toutes ces variable sont des variables de sessions (à l'exception de 
`max_worker_processes`), ce qui signifie qu'elles reprennent, lors 
d'une nouvelle connexion, les valeurs qui leur sont attribuées dans le 
fichier de `postgresql.conf`ou leurs 
valeurs par défaut si elles n'y sont pas définies. Une fois définies 
en session, les nouvelles valeurs sont conservées jusqu'à déconnexion du 
client ou jusqu'au moment où elle seront redéfinies.
-->


Il est donc possible de faire du cas par cas en appliquant des valeurs différentes pour une requête gourmande lors d'une plage horaire avec un trafic moindre.
`SET` est par défaut valable pour la `session`.
On peut, le cas échéant, utiliser `SET LOCAL` pour limiter ce paramétrage à la `transaction` en cours.

## Derniers conseils et mises en garde

**Rappel** 

Comme je l'ai déjà mentionné la fois précédente, pour un bon **réglage de départ de l'instance**, je vous conseille de partir sur
[`max_parallel_workers_per_gather`](https://www.postgresql.org/docs/current/static/runtime-config-resource.html#GUC-MAX-PARALLEL-WORKERS-PER-GATHER)
égalant le nombre de cœurs (virtuels ou non).
Vous pouvez régler [`max_parallel_workers`](https://www.postgresql.org/docs/current/static/runtime-config-resource.html#GUC-MAX-PARALLEL-WORKERS) avec une valeur égale ou inférieure à
[`max_worker_processes`](https://www.postgresql.org/docs/current/static/runtime-config-resource.html#GUC-MAX-WORKER-PROCESSES) dans le fichier de configuration `postgresql.conf` **uniquement**.

**Aller plus loin**
<!-- JCA
Mériterait une entrée TOC? --ETA: Trouble obsessionnel compulsif ? Table of Contents ? 
-->

Afin de tirer davantage de la parallélisation, voici quelques principes élémentaires&nbsp;:

<!-- JCA
Peut être regrouper ce qu'on doit changer à la session au début, et 
ce qu'on doit changer au niveau de postgresql.conf ensuite ?
-->

* Pour **forcer la parallélisation**, `force_parallel_mode` à `ON`. Mais dans la plupart des cas _c'est totalement inutile_.

* Pour **déclencher la parallélisation** d'une requête, il peut aussi être indiqué de dimininuer `parallel_tuple_cost` (même en dessous de `cpu_tuple_cost`) 
et baisser `parallel_setup_cost` à 100 (par exemple) si vous constatez comme moi, que celui-ci semble trop élevé avec à la clé, de bonnes surprises. 
Mais surtout, si vous êtes amené à ajuster les paramètres de coût en général, je pense qu'il faut ajouter celui-ci à la liste. 
D'autant plus si votre matériel est performant (fréquence CPU élevée, disque SSD...).
Le conseil&nbsp;: si vous modifiez les coûts, modifiez alors le coût d'un facteur identique (e.g. si vous divisez par 2 `cup_tuple_cost` faites de même sur `parallel_tuple_cost/setup`).
Pour le coût de lancement (setup), vous pouvez démarrer avec une valeur identique au temps réel donné par EXPLAIN ANALYZE. 
Les unités diffèrent (l'un est arbitraire et l'autre le temps réel en ms), cependant il faut garder à l'esprit que les valeurs par défaut correspondent à du matériel physique standard (HDD).

<!-- JCA
Il serait fort intéressant de dire pourquoi on doit descendre et 
monter ces valeurs, car dans ce cas, on ne voit pas trop les use case 
pour les modifications... Je n'ai pas de cas concret à te poser
-->

* Pour **désactiver totalement la paralléllisation**, il suffit de placer `max_parallel_workers_per_gather = 0` si le besoin s'en fait sentir 
(_e.g._ vous constatez que c'est bénéfique sur un plan d'exécution ou toute autre raison).

* Pour **augmenter la parallélisation**, augmenter `max_parallel_workers_per_gather`, mais aussi `max_parallel_workers` en fonction du nombre de CPU ou coeurs disponibles.

* Ne modifier les autres paramètres que lors d'une **session** (ou **transaction**) pour ne pas parasiter le comportement de toutes les requêtes de votre instance.

* **Expérimentez** dans un **environnement de test** ou sur un **réplica**.

Toutefois, attention à un trop grand degré de parallélisation qui peut consommer d'avantage de mémoire et de CPU qu'attendu, ce qui pourrait engendrer des résultats contre-productifs
(voir le résultat pour les agrégats, ceci est d'ailleurs mentionné dans la documentation officielle).

Ces optimisations ont leur limites.
On ne peut pas "imposer" à l'optimiseur de faire une requête en parallèle en utilisant seulement `force_parallel_mode` (comme nous l'avons vu plus haut),
mais on peut quand même l'amener à faire ce choix avec de bonnes (ou mauvaises) surprises.

C'est tout pour cette fois, mais rendez-vous pour la dernière partie qui pourrait bien vous surprendre.


Another day in Parallel&nbsp;!

