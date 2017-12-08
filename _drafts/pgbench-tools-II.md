---
layout: post
Title: Pgbench-tools - Going further
Draft: true
published: true
---
      
This is a translation from french. You can find the original post [here](http://www.loxodata.com/post/benchmarking-pratique2/)

Last time I spoke of pgbench-tools a benchmarking automatization tool.
Today we are going to see how to make a proper start and give an example.
Let's dive deeper into this tool !

<!-- ## Déroulement d'une série de tests - étape par étape -->
## Unravelling the tests - step by step


We are going to take a look at what you can expect with **./runset** which starts a series of tests within a defined "set".
Everything that follows i scripted.


### DB Intialisation 


The first stop is to create a database, here default `pgbench_` which works the same as&nbsp;:

    pgbench -i

### Test

<!--pgbench est ensuite "lancé" en mode test avec le nombre de clients listés dans le fichier de configuration. 
S'il y a plusieurs valeurs pour le nombre de clients dans le fichier **config**, une boucle est effectuée.-->
Then pgbench is launched just like a regular bench with a previously set number of clients from the `config` file.
If it happens to have more than one value for the clients number in that file, a loop is performed.
A second loop is performed within if the file contains several values for scales (please refer to the previous post for details).

### Data collection

Among data collection one can find:

 * speed (tps) and total amount of transactions;
 * `latency` (average, maximum and 90th percentile): the 5 worst latency values are displayed on screen for information purpuses;
 * `checkpoints`&nbsp;: le nombre de `checkpoints` effectués pendant le test&nbsp;;
 * the number of checkpoints performed during the test;
 * metrics regarding buffers `buf_check, buf_clean, buf_backend, buf_alloc, max_clean, backend_sync` are all explained [here](https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-BGWRITER-VIEW).
As well as `max_dirty` and `wal_written`.


### Global report

<!--Un fichier `index.html` contenant le rapport général est créé à la racine du dossier avec les graphiques précédemment cités (tps en fonction du nombre de client et du facteur d'échelle).
Des tableaux comparatifs pour chaque set sont également produits pour simplifier la consultation.
Pour chaque test de chaque set, un rapport `index.html` est de plus généré avec les graphiques de tps, latence, iostat, meminfo, vmstat dans le dossier `result/numérodutest` (pour observer l'évolution des métriques au cours test).--> 

An `index.html` file is created at the root of the folder. It contains the aformentionned graphs (tps against client number and scaling factor).
Tables containing detailled results for different sets are produced to simplify reading.
For each test of each set, another report also named `index.html` is generated with graphs of tps, latency, iostat, meminfo and vmstat in a sub-folder `result/numberofset` which can give very extended information regarding the behaviour of server during the test.

### Example

Here is what to expect in your shell after firing  `runset`:
~~~

postgres@BENCHER pgbench-tools-master]$ ./runset
Removing old pgbench tables
DROP TABLE
VACUUM
Creating new pgbench tables
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
creating tables...
100000 of 100000 tuples (100%) done (elapsed 0.03 s, remaining 0.00 s)
vacuum...
set primary keys...
done.
Run set #1 of 3 with 1 clients scale=1
Running tests using: psql -h localhost -U postgres -p 5432 -d pgbench
Storing results using: psql -h localhost -U postgres -p 5432 -d results
Found standard pgbench tables with prefix=pgbench_
Cleaning up database pgbench
TRUNCATE TABLE
VACUUM
CHECKPOINT
Waiting for checkpoint statistics
INSERT 0 1
This is test 2267
Script tpc-b_96.sql executing for 1 concurrent users...
./benchwarmer: line 249: 19127 Killed                  ./timed-os-stats vmstat > results/$TEST/vmstat.log  (wd: /var/postgres/pgbench-tools-master)
./benchwarmer: line 249: 19128 Killed                  ./timed-os-stats iostat > results/$TEST/iostat.log  (wd: /var/postgres/pgbench-tools-master)
./benchwarmer: line 249: 19129 Killed                  ./timed-os-stats meminfo > results/$TEST/meminfo.log  (wd: /var/postgres/pgbench-tools-master)
UPDATE 1
transaction type: /var/postgres/pgbench-tools-master/tests/tpc-b_96.sql
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 60 s
number of transactions actually processed: 32095
latency average = 1.869 ms
tps = 534.911951 (including connections establishing)
tps = 535.018561 (excluding connections establishing)
Cleaning up database pgbench
TRUNCATE TABLE
VACUUM
CHECKPOINT
Worst latency results:
43635
43909
45061
45594
1042786

~~~

### Report for each test


Last time I wrote about the __set report__, but this time I will focus on the __test report__.
Collected data are stored, sorted and grouped not only by set but also by test, in the aformentionned database which is declared in the config file (defaults `results`).


<!--Je vous avais déjà parlé dans l'article précédent du contenu du __rapport de set__, je vais détailler un peu plus ici le contenu du __rapport de test__.
Les donneés collectées sont stockées, ordonnées et groupées par set et par test, dans la base déclarée dans le fichier de configuration (par défaut `results`).-->


One can find a subfolder named after each test's number in the main folder `pgbench-tools/results`. 
One can easily copy these without having to make any dump from de database itself.

<!--On peut retrouver dans le dossier `pgbench-tools/results` un dossier pour chaque test dont le nom est le numéro du test et qui comporte ces mêmes informations.
Cela permet de copier facilement les résultats de bench sans extraction supplémentaire.-->

At last, metrics such as `vmstat`, `iostat` and `meminfo` are collected into logfile in this very same folder.
One can also find graphs showing the behaviour of tps, latency, CPU, memory, buffers and much more for each test.
We will come back to this matter and go into more details latter on.
Here is a sneak peak of graphs taken randomly.

<!-- Enfin, la collecte des métriques `vmstat`, `iostat` et `meminfo` est ajoutée sous forme de fichiers logs dans ce même répertoire.
On y trouve également des graphiques montrant l'évolution au cours du temps lors du test&nbsp;:  tps, latence, cpu, mémoire, buffers...
Nous reviendrons avec encore davantage de détail sur ce sujet ultérieurement.
Voici un aperçu des graphiques pour un test pris au hasard.-->

<table style="border: 0px;">
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/tps.png" alt="tps.png : profil des tps au cours du test" style="width: 500px; border: 0px;"/></td>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/latency.png" alt="latency.png latence au 9e decile" style="width: 500px; border: 0px;"/></td>  
  </tr>
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/iostat-writeMB.png" alt="iostat-writeMB.png" style="width: 500px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/iostat-util.png" alt="iostat-util.png" style="width: 500px; border: 0px;"/></td>
  </tr>
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/vmstat.png" alt="vmstat.png" style="width: 500px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/dirty.png" alt="dirty.png :  buffers sales " style="width: 500px; border: 0px;"/></td>
  </tr>
</table>


## Parameters in details


### Foreword on scale, clients and the scripts used

You may find relevant information to help you further on theoretical aspect of scale on this [wiki](https://wiki.postgresql.org/wiki/Pgbenchtesting).

<!--Vous trouverez des informations qui permettent d'appréhender la partie théorique sur le facteur d'échelle sur ce [wiki](https://wiki.postgresql.org/wiki/Pgbenchtesting).-->

#### SCALE 

Since rows are produced randomly, their content and structure is of little consequence because it is merely database taken as an example.
What needs to be understood here, is the size which is going to be taken at different scales is going to fit differently in cache, memory and disk.
On a server with 8G of RAM and 20M cache, here is what one can expect :

<!-- Les lignes étant produites aléatoirement, leur contenu et leur structure n'ont ici pas ou peu d'importance car il s'agit d'une base témoin.
Ce qu'il faut retenir c'est la taille que va occuper la base aux différentes échelles.
Sur une machine avec 8Go de RAM et 20Mo de cache voici comment la base de données peut être répartie (globalement)&nbsp;:
Sur une machine avec 8Go de RAM et 20Mo de cache voici comment la base de données peut être répartie (globalement)&nbsp;: -->

| facteur d'échelle  | taille correspondante | répartition des données  |
| scale  | matching size | data main location  |
|--------------------|-----------------------|--------------------------|
| 1 |  15Mo | **shared_buffers** |
| 10 | 150Mo | **shared_buffers** |
| 100 | 1500Mo | **shared_buffers/RAM** |
| 1000 | 15Go | **RAM/Disk** |
| 10000 | 150Go | **Disk**  |

Here I kept the defaults **SCALES="1 10 100 1000"**

#### Clients

Clients are simultaneus connexions expected on the cluster. 
I kept the defaults **SETCLIENTS="1 2 4 8 16 32"**.


#### Type of SQL script
The type of `sql` script also of much importance. Here is a list of what is available in the package&nbsp;:

* __select__&nbsp;: Contains a transaction with a random SELECT on the PK. Perfect for replicas in Read-Only mode&nbsp;;
* __insert__&nbsp;: INSERT of a random content&nbsp;;
* __insert-size__&nbsp;: Bulk INSERT with the number of lines as parameter of input&nbsp;;
* __update__&nbsp;: A single transaction with an UPDATE of random value.

These last three transactions are fit for a primary which might suffers writes in different contexts.

Remains mixed use cases for single clusters&nbsp;:

* __nobranch__&nbsp;: A transaction with a SELECT, an UPDATE and an INSERT&nbsp;;
* __tpc-b__ like: uses the standard [tpc](https://en.wikipedia.org/wiki/Transaction_Processing_Performance_Council) or the source site [tpc-b](http://www.tpc.org/tpcb/default.asp).
Which consists of 3 UPDATEs, 1 SELECT, 1 INSERT. This is the default and I kept it. It is very broad and  covers much use cases.

This choice is paramount and must be chosen carefully having in mind which one is relevant to you.
Size of database must be taken into account, concurrent client connexions, architecture.
In my case, clusters range from few MB to 1,9TB and clients rarely exeeds 32.
Most instances are under the Database Administrators responsibility and are used in mixed case scenario with a single cluster.
Therefore the choice of keeping much of the defaults makes perfect sens **in my case**.

<!-- Ces choix doivent donc être faits en fonction du cas pratique qui vous occupe. 
Il faut prendre en compte la taille de votre base de données, le nombre de connexions simultanées ainsi que le type d'architecture que vous employez en production.
Dans mon cas, les instances sont de tailles allant de quelques Mo à 1,5To et le nombre de client doit rarement dépasser 32.
La plupart des instances sous la responsabilite du pôle SGBD dont je fais partie sont à instance unique en utilisation mixte.
Le choix de conserver les valeurs par défaut a donc tout son sens dans mon cas.-->

## A word about outliers

In order to be rid of the background noise, free from artefacts, tests must be carried out throughout a timeframe long enough (a few minutes if possible). 
To mitigate this issue, it is also possible to run the same test several times. 
This way one can get an average value.

By seperating the outliers, one can obtain average values that represents a behaviour we might call "normal". If a test fails it is advised to delete it.

<!-- Nous cherchons avant tout à lisser le bruit de fond, à obtenir des mesures exemptes de parasites.
Dans ce but, les tests doivent être assez longs (plusieurs minutes si possible).
Pour pallier ce problème, ou en diminuer d'avantage l'importance, il est également possible de reconduire le même test plusieurs fois. Ceci permet donc de faire une moyenne.

En séparant les valeurs aberrantes, on obtient des moyennes plus représentative du comportement dit normal. Si un test est en erreur (problème lors du bench), il est alors conseillé de le supprimer.-->


### Parameters used

For my first round of tests, I kept the defaults values that fit my need?

#### Test related parameters

* **SETTIMES=3**&nbsp;: This is the number of times the test is run (in order to mitigaite the background noise _e.g._ unexpected checkpoints&nbsp;;
* **RUNTIME=60**&nbsp;: This is the duration of a single test (option -T). It might be deemed appropriate to extend it according to the frequency of checkpoints. These have a very important effect on writes as well as reads. It is most interresting to have one or more checkpoint during a test.
However the performing several times the same test in a non back-to-back cycle tends to alleviate this effect.

For the first part of my test, TOTTRANS (total number of transactions) and SETRATES (target number of tps) were sidelined.


#### Disk related parameters
* **OSDATA=1**&nbsp;: Triggers the data collection of vmstat and iostat
* **DISKLIST="sda"**&nbsp;: The choice of the device on which statistics are collected


#### Miscellaneous parameters
* **report-latencies**&nbsp;: Average latency per statement 
* **MAX_WORKERS="auto"**&nbsp;: I strongly advise keeping this one to defaults unless testing parallel query itself

## VACUUM

Quick calculation will give you 4 scales (**1, 10, 100, 1000**), 6 values for clients (**1, 2, 4, 8, 16, 32**) and 3 test for each combination (**SETTIMES=3**) during 1 minute. 
The total is therefore 4*6*3=72 tests of 1 minute. 
Initial loading of the database not taken into account.

Following the MVCC (management of multiple versions of a single tuple resulting of delete and update), all the transaction are going to leave dead tuples.
Moreover, when many rows are updated (or deleted), it is necessary update the statistics as well to insure the planner always takes the best plan for every query.

To avoid autovacuum in between tests and slowing of performance, a `VACUUM ANAYZE` is performed if no vacuum has been done in last 10 seconds of cleanup phase of the code during the last test.

It is an approximation of course, but the downside is having to many VACUUM instead of to little. 
It is a sensible argument for anyone careful about regular VACUUM.

<!-- En faisant un rapide calcul, on constate que 4 facteurs d'échelles pour les données (**1, 10, 100, 1000**), 6 échelles différentes pour les clients (**1, 2, 4, 8, 16, 32**), lancées 3 fois (**SETTIMES=3**) 
pour chaque couple de paramètre pendant 1 minute nous donne un set total de&nbsp;:
4*6*3= 72 tests d'une minute. Ceci ne tenant évidemment pas compte du chargement de la base. -->

<!-- Suite à la gestion des différentes versions d'une ligne (MVCC), une transaction commitée laisse des lignes mortes. 
De plus, lorsqu'un grand nombre de données est modifié, il est nécessaire de mettre à jour les statistiques pour s'assurer que l'optimiseur prend toujours le meilleur chemin pour effectuer la requête. -->

<!-- Pour éviter qu'un autovacuum lié aux modifications du test précédant ne viennent ralentir le test actuel, un `vacuum analyze` est lancé si aucun vacuum n'a été réalisé pendant les 10 dernières secondes du dernier test lors de la phase "cleanup" du code. -->

<!-- Il s'agit ici d'une approximation, mais le revers est d'avoir trop de VACUUM plutôt que pas assez.
C'est un argument valide si on est habituellement vigilant sur les VACUUM réguliers.-->


## Conclusion

Vacuum, reproducibility, serialization and OS/DB statistics turn pgbench-tools into an extraordinary wrapper of pgbench.
It is possible to find relevant tests for many different use cases according to the type of test, the size of the dataset, the number of clients among other things (remaining as approximation of course).

<!-- Le vacuum, la reproductibilité, la sérialisation et les statistiques font de pgbench-tools une surcouche d'industrialisation de pgbench déjà extraordinaire.
Il est possible de trouver les tests pertinents pour de nombreux cas pratiques (ceux-ci restent des approximations bien sûr) en fonction du type de test, du volume de données, du nombre de connexions concurrentes entre autres choses.-->

However this tool has some limitations which we are going to explore in a next - _more practical_ - article with a complete series of test and a context set in the real world with its own constraints.
I will also speak about many other aspects : interpretation of the results, the strategy underneath, and how to reach a (relatively) definitive conclusion.
I will count the changes, improvements, corrections and new features on this project and they are numerous.

Until next time, keep benching guys !

<!-- Cependant, cet outil comporte des limitations que nous allons aborder dans un prochain article plus concret avec une série de test complète et un cadre de départ avec ses propres contraintes lié au contexte.
J'aborderai également de nombreux aspects : interprétation des données statistiques, comment diriger une campagne de test, comment faire converger les résultats obtenus vers une conclusion.
Je passerai aussi en revue les changements, améliorations, corrections et ajouts qu'il m'a été donné de contribuer sur ce projet et ils sont nombreux.-->


<!-- Just bench it !-->
