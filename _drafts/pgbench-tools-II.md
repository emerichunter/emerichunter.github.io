---
layout: post
Title: Pgbench-tools - Going further
Draft: true
published: true
---      
<!--
Je vous ai parlé la [dernière fois](/post/benchmarking-pratique/) de pgbench-tools, un outils d'automatisation de test de bench. 
Je vous propose aujourd'hui de démarrer avec un exemple et de plonger plus profondément dans cet outils.
-->

Last time I spoke of pgbench-tools, which is benchmarking automatization tool.
Today we are going to see how to start and give an example.
Let's dive deeper into this tool !

<!-- ## Déroulement d'une série de tests - étape par étape -->
## Unravelling the tests - step by step

<!-- Voici ce à quoi vous pouvez vous attendre lorsque vous lancez pgbench-tools avec **./runset** 
vous lancez une série de test appelée "set".
Tout ce qui suit est scripté. -->
We are going to take a look at what you can expect with **./runset** which starts a series of tests within a defined "set".
Everything that follows i scripted.

<!-- ### Initialisation de la base-->
### DB Intialisation 

<!--Il s'agit de l'étape permettant de créer des données dans la base.
L'outils réalise cette étape seul, en fonction du paramètre `scale`, il correspond à la commande suivante&nbsp;:
-->
The first stop is to create a database, here default `pgbench_` which works the same as&nbsp;:

    pgbench -i

### Test

pgbench est ensuite "lancé" en mode test avec le nombre de clients listés dans le fichier de configuration. 
S'il y a plusieurs valeurs pour le nombre de clients dans le fichier **config**, une boucle est effectuée.
Then pgbench is launched just like a regular bench with a previously set number of clients from the `config` file.
If it happens to have more than one value for the clients number in that file, a loop is performed.
A second loop is performed if the file contains several values for scales (please refer to the previous post for details).

### Collecte des données
Les données collectées comportent entre autres&nbsp;:

 * `transactions`&nbsp;: la vitesse (`tps`) et le nombre total de transactions&nbsp;;
 * `latence` (moyenne, maximum et neuvième décile)&nbsp;: les 5 plus mauvais temps de latence sont affichées à l'écran pour information&nbsp;;
 * `checkpoints`&nbsp;: le nombre de `checkpoints` effectués pendant le test&nbsp;;
 * `buffers`&nbsp;: les métriques `buf_check, buf_clean, buf_backend, buf_alloc, max_clean, backend_sync` sont expliquées [ici](https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-BGWRITER-VIEW).
Mais aussi `max_dirty` et `wal_written`.


### Génération du rapport
Un fichier `index.html` contenant le rapport général est créé à la racine du dossier avec les graphiques précédemment cités (tps en fonction du nombre de client et du facteur d'échelle).
Des tableaux comparatifs pour chaque set sont également produits pour simplifier la consultation.
Pour chaque test de chaque set, un rapport `index.html` est de plus généré avec les graphiques de tps, latence, iostat, meminfo, vmstat dans le dossier `result/numérodutest` (pour observer l'évolution des métriques au cours test).

### Exemple

Voici ce qu'il se passe lorsqu'on lance la commande `runset`.

~~~

postgres@NOEYYRYH pgbench-tools-master]$ ./runset
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

### Rapport pour chaque test

Je vous avais déjà parlé dans l'article précédent du contenu du __rapport de set__, je vais détailler un peu plus ici le contenu du __rapport de test__.
Les donneés collectées sont stockées, ordonnées et groupées par set et par test, dans la base déclarée dans le fichier de configuration (par défaut `results`).

On peut retrouver dans le dossier `pgbench-tools/results` un dossier pour chaque test dont le nom est le numéro du test et qui comporte ces mêmes informations.
Cela permet de copier facilement les résultats de bench sans extraction supplémentaire.

Enfin, la collecte des métriques `vmstat`, `iostat` et `meminfo` est ajoutée sous forme de fichiers logs dans ce même répertoire.
On y trouve également des graphiques montrant l'évolution au cours du temps lors du test&nbsp;:  tps, latence, cpu, mémoire, buffers...
Nous reviendrons avec encore davantage de détail sur ce sujet ultérieurement.
Voici un aperçu des graphiques pour un test pris au hasard.

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

## Détail des paramètres

### Un mot sur l'échelle, le nombre de clients et le type de scripts

Vous trouverez des informations qui permettent d'appréhender la partie théorique sur le facteur d'échelle sur ce [wiki](https://wiki.postgresql.org/wiki/Pgbenchtesting).

#### SCALE (échelle)
Les lignes étant produites aléatoirement, leur contenu et leur structure n'ont ici pas ou peu d'importance car il s'agit d'une base témoin.
Ce qu'il faut retenir c'est la taille que va occuper la base aux différentes échelles.
Sur une machine avec 8Go de RAM et 20Mo de cache voici comment la base de données peut être répartie (globalement)&nbsp;:

| facteur d'échelle  | taille correspondante | répartition des données  |
|--------------------|-----------------------|--------------------------|
| 1 |  15Mo | **shared_buffers** |
| 10 | 150Mo | **shared_buffers** |
| 100 | 1500Mo | **shared_buffers/RAM** |
| 1000 | 15Go | **RAM/Disque** |
| 10000 | 150Go | **Disque**  |

J'ai laissé ce paramètre par défaut **SCALES="1 10 100 1000"**

#### Nombre de clients

Cela dépend donc du nombre de connexions simultanées que vous attendez sur votre instance.
J'ai laissé ce paramètre par défaut **SETCLIENTS="1 2 4 8 16 32"**.

#### Type de script SQL
Le type de script `sql` est également très important. Voici la liste disponible&nbsp;:

* __select__&nbsp;: contient une transaction avec un select sur valeur de clé primaire aléatoire. Parfait pour simuler une transaction sur un réplica en lecture seule&nbsp;;
* __insert__&nbsp;: insertion d'une ligne de valeur aléatoire&nbsp;;
* __insert-size__&nbsp;: insert massif avec en paramètre d'entrée le nombre de lignes&nbsp;;
* __update__&nbsp;: transaction avec une mise a jour sur ligne unique de valeur aléatoire.

Ces trois dernières transactions conviennent pour un nœud primaire qui reçoit les écritures dans différents contextes d'utilisation.

Pour les utilisations mixtes sur une instance unique&nbsp;:

* __nobranch__&nbsp;: une transaction comportant un select, un update, un insert&nbsp;;
* __tpc-b__ like: caractéristique des transactions [tpc](https://en.wikipedia.org/wiki/Transaction_Processing_Performance_Council) ou sur le site original [tpc-b](http://www.tpc.org/tpcb/default.asp).
Qui consiste en 3 updates, 1 select, 1 insert. C'est le paramètre par défaut que j'ai choisi. Il est générique et permet de couvrir une large palette de cas d'utilisation.

Ces choix doivent donc être faits en fonction du cas pratique qui vous occupe. 
Il faut prendre en compte la taille de votre base de données, le nombre de connexions simultanées ainsi que le type d'architecture que vous employez en production.
Dans mon cas, les instances sont de tailles allant de quelques Mo à 1,5To et le nombre de client doit rarement dépasser 32.
La plupart des instances sous la responsabilite du pôle SGBD dont je fais partie sont à instance unique en utilisation mixte.
Le choix de conserver les valeurs par défaut a donc tout son sens dans mon cas.

## Un mot sur les valeurs aberrantes

Nous cherchons avant tout à lisser le bruit de fond, à obtenir des mesures exemptes de parasites.
Dans ce but, les tests doivent être assez longs (plusieurs minutes si possible).
Pour pallier ce problème, ou en diminuer d'avantage l'importance, il est également possible de reconduire le même test plusieurs fois. Ceci permet donc de faire une moyenne.

En séparant les valeurs aberrantes, on obtient des moyennes plus représentative du comportement dit normal. Si un test est en erreur (problème lors du bench), il est alors conseillé de le supprimer.

### Paramètres utilisés
Pour les premiers tests, j'ai conservé les valeurs par défaut du fichier qui sont assez bien pensées.
#### Paramètres relatifs aux tests
* **SETTIMES=3**&nbsp;: il s'agit du nombre de fois ou le test est reproduit (ceci permet de lisser le bruit de fond e.g. les checkpoints inopinés)&nbsp;;
* **RUNTIME=60**&nbsp;: c'est la durée d'un test individuel (option -T). Il peut convenir de l'allonger en fonction de la fréquence des checkpoints.
Ceux-ci ont un impact important en écriture (mais aussi en lecture). Il est intéressant d'inclure un ou plusieurs checkpoints dans un bench.
Cependant, le fait de réaliser plusieurs fois le même test de façon cyclique et non consécutive à pour effet de gommer cette importance.

Je n'ai pas utilisé les paramètres TOTTRANS (nombre total de transactions) et SETRATES (taux de transaction par seconde cible qui est déclaré) pour la première partie de mes tests.

#### Paramètres relatifs aux disques
* **OSDATA=1**&nbsp;: Enclenche la collection des données issues de vmstat et iostat
* **DISKLIST="sda"**&nbsp;: Le choix du disque sur lequel les statistiques sont collectées

#### Autres paramètres
* **report-latencies**&nbsp;: Latence moyenne par déclaration (statement)
* **MAX_WORKERS="auto"**&nbsp;: je conseille de laisser ce paramètre par défaut à moins de bencher la parallèlisation (-j)

## VACUUM

En faisant un rapide calcul, on constate que 4 facteurs d'échelles pour les données (**1, 10, 100, 1000**), 6 échelles différentes pour les clients (**1, 2, 4, 8, 16, 32**), lancées 3 fois (**SETTIMES=3**) 
pour chaque couple de paramètre pendant 1 minute nous donne un set total de&nbsp;:
4*6*3= 72 tests d'une minute. Ceci ne tenant évidemment pas compte du chargement de la base.

Suite à la gestion des différentes versions d'une ligne (MVCC), une transaction commitée laisse des lignes mortes. 
De plus, lorsqu'un grand nombre de données est modifié, il est nécessaire de mettre à jour les statistiques pour s'assurer que l'optimiseur prend toujours le meilleur chemin pour effectuer la requête.

Pour éviter qu'un autovacuum lié aux modifications du test précédant ne viennent ralentir le test actuel, un `vacuum analyze` est lancé si aucun vacuum n'a été réalisé pendant les 10 dernières secondes du dernier test lors de la phase "cleanup" du code.

Il s'agit ici d'une approximation, mais le revers est d'avoir trop de VACUUM plutôt que pas assez.
C'est un argument valide si on est habituellement vigilant sur les VACUUM réguliers.


## Conclusion

Le vacuum, la reproductibilité, la sérialisation et les statistiques font de pgbench-tools une surcouche d'industrialisation de pgbench déjà extraordinaire.
Il est possible de trouver les tests pertinents pour de nombreux cas pratiques (ceux-ci restent des approximations bien sûr) en fonction du type de test, du volume de données, du nombre de connexions concurrentes entre autres choses.

Cependant, cet outil comporte des limitations que nous allons aborder dans un prochain article plus concret avec une série de test complète et un cadre de départ avec ses propres contraintes lié au contexte.
J'aborderai également de nombreux aspects : interprétation des données statistiques, comment diriger une campagne de test, comment faire converger les résultats obtenus vers une conclusion.
Je passerai aussi en revue les changements, améliorations, corrections et ajouts qu'il m'a été donné de contribuer sur ce projet et ils sont nombreux.


Just bench it !
