---
published: false
---
# FAILOVER

<!-- 3e Réécriture&nbsp;: -->

Today, I am going to talk about automatic failover.

Not just about cluster failvover (primary failure and standby taking over) but from client point of view.

Yes, PostgreSQL 10 is out since october 5th and it has (among others) a new awesome feature. 
Client can reconnect automaticaly to cluster after failure to either RO or RW instance.

So, here are some questions you might have :
Is it easy to setup ? 
How much downtime ?
Above all else : is the damn thing working ?

## In the Beginning... 


### Which HA ?

To cut it short I chose `REPMGR`.

Configuration is 3 nodes (1 primary and 2 standbys) plus a witness server.
In case you wonder, this last player serves as as casting vote in a situation of doubt over which node need to be promoted.

![Configuration REPMGR](https://www.loxodata.com/images/post/failover-2017/config_REPMGR.png)


Here is an extract from repmgr.conf&nbsp;:

~~~
cluster=test_repmgr
node=4
node_name=node4
conninfo='port=5432 host=server1 user=repmgr dbname=repmgr'
failover=automatic
promote_command='repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='repmgr standby follow -f /etc/repmgr.conf --log-to-file'
                 # the server's hostname or another identifier unambiguously
                 # associated with the server to avoid confusion
logfile='/var/log/repmgr/repmgr-9.6.log'
pg_bindir=/usr/pgsql-9.6/bin/

master_response_timeout=5
reconnect_attempts=3
reconnect_interval=5
retry_promote_interval_secs=10

~~~

Here is what's expected:</br>
`Downtime` = `master_response_timeout` + `reconnect_attempts`*`reconnect_interval`, </br>
Roughly 5 + 3*5 = 20 secondes of downtime for the database.


### A man who is his own lawyer...


Once configuration is setup, how can one measure downtime ?
What needs to be done to reconnect the client to the new primary as soon as it is available ?

The solution is [automatic client failover](https://wiki.postgresql.org/wiki/New_in_postgres_10#Connection_Failover_and_Routing_in_libpq).

This is entirely about the client. The clusters are all in version 9.6.
It is important to notice that in order to benefit from this feature it is not necessary to install PG10 on the clusters, older versions will do just fine

The client is only looking for the first node available.
**READ-ONLY** (RO): It will pick any node (primary or secondary).
**READ-WRITE** (RW): It will pick the first writable node (primary).

Implications are important, with a solution to automatically failover connections, it is not longer necessary to rewrite the connection string manually.
The client is going to regain a connection and be instrumental in the measurement of downtime.

#### Figuring how to setup the connection string properly

Let us do a little  bit of reading [here](http://paquier.xyz/postgresql-2/postgres-10-libpq-read-write/) and [there](http://paquier.xyz/postgresql-2/postgres-10-multi-host-connstr/). This will allow us to get a pretty good idea on how to proceed.
Essentially:

* We need to indicate wether we need to connect on a primary (RW) or on a any cluster (RO)
* Il faut préciser dans la chaîne de connexion si l'on souhaite se connecter à une instance en lecture-écriture (primaire) ou
une instance sans distinction lecture seule ou lecture-écriture (indifféremment standby ou primaire).
<!-- SAS&nbsp;: pas clair du tout. Que veut-on mesurer ? -->
En l'occurrence, pour un failover des écritures, il me faut retrouver une connexion sur un primaire acceptant les écritures.

* Mais ce n'est pas tout, il faut inclure la liste des nœuds qui composent notre configuration, avec les ports.

<!-- /SAS -->

<!-- LAV: sur le paragraphe comment, tu dois vraiment expliquer le comment. Là,
ça donne l'impression que tu renvoies sur un autre blog pour l'explication...
Alors que ce n'est pas le cas parce que tu donnes un exemple pratique en dessous!-->

**Cas pratique**

Voici un petit extrait du script que j'ai écrit en suivant les indications données (et en tâtonnant un peu) et qui vous aiguillera encore d'avantage sur la façon de procéder&nbsp;:

~~~

TIME_RES=10000 # time resolution in µs

MONITORING_DB=monitoring_replication


PORT_1=5432
PORT_2=$PORT_1
PORT_3=$PORT_1

HOST_1=server1
HOST_2=server2
HOST_3=server3

CONNINFO="postgresql://"$HOST_1":"$","$HOST_2":"$PORT_2","$HOST_3":"$PORT_3"/"$MONITORING_DB"?target_session_attrs=read-write"

~~~

Une fois la reconnexion établie, je vais pouvoir mesurer le temps d'indisponibilité total et quantifier en terme de temps les écritures perdues pendant l'opération de failover.


#### Quantifier l'indisponibilité&nbsp;: Comment&nbsp;?

J'ai utilisé usleep pour écrire dans une table de log toute les 10ms ([voir note pour davantage de détail](#note)) en utilisant la chaîne de connexion décrite plus haut.

Voici le lien vers l'utilitaire employé pour ce test&nbsp;: [monitoring_replication](https://github.com/emerichunter/monitoring_replication).
Il m'a permis de mesurer le temps pendant lequel le cluster était indisponible durant la panne et le failover.
L'installation et le mode d'emploi sont expliqués en détails dans le README.

<!--LAV: Le pourquoi n'est pas le pb dans cet article censé présenter une nouvelle
fonctionnalité de postgres 10. Dis simplement que tu échantillonnes toutes les 10ms
sans expliquer pourquoi et rajoute en aparté le lien vers ton outil (ou alors j'ai mal compris) -->

## Remplissons le WAL&nbsp;!

<!--LAV : il faut que tu expliques que tu cherches à provoquer une panne pour tester
le failover... Le remplissage des WALs n'est peut-être pas la meilleure panne à tester.
En effet, de manière habituelle les file systems sont de même taille sur le primaire
et le secondaire... Tu risques donc d'avori la même panne sur les 2 instances.

Tu peux utiliser pg_crash pour tester une panne. Ou sinon explique simplement que
tu crées cette panne (remplissage des WALs) de manière artificielle parce que la
teneur de la panne n'a aucune importance, c'est le fait qu'il y ait une panne
qui est important.-->

Pour tester notre bascule, il faut provoquer une panne sur le primaire.
Une panne facile à provoquer est la saturation du FS par une grande quantité de fichiers WAL.
Une fois la panne provoquée et le failover de base de données effectuée, nous regardons ensuite dans les traces laissées par repmgrd.log puis dans la table de log créée et approvisionnée par notre utilitaire.

<!-- SAS&nbsp;: Ca sort d'où ? -->
Voici un extrait de la trace log de `repmgrd.log` correspondant au test de remplissage de FS.
La détection de la panne a lieu ligne 2.
Les trois dernières lignes correspondent à la promotion du standby choisi en primaire.
Notez que le marqueur de temps comptabilise 21 secondes jusqu'à la promotion du standby en nouveau primaire.

~~~~

log node 2 (standby failover)
[2017-07-13 10:29:25] [ERROR] connection to database failed: could not connect to server: Connection refused
        Is the server running on host "server1" (172.134.11.45) and accepting
        TCP/IP connections on port 5432?

[2017-07-13 10:29:25] [ERROR] unable to connect to upstream node: could not connect to server: Connection refused
        Is the server running on host "server1" (172.134.11.45) and accepting
        TCP/IP connections on port 5432?

[2017-07-13 10:29:25] [ERROR] connection to database failed: could not connect to server: Connection refused
        Is the server running on host "server1" (172.134.11.45) and accepting
        TCP/IP connections on port 5432?

[2017-07-13 10:29:25] [WARNING] connection to master has been lost, trying to recover... 15 seconds before failover decision
[2017-07-13 10:29:30] [WARNING] connection to master has been lost, trying to recover... 10 seconds before failover decision
[2017-07-13 10:29:35] [WARNING] connection to master has been lost, trying to recover... 5 seconds before failover decision
[2017-07-13 10:29:40] [ERROR] unable to reconnect to master (timeout 5 seconds)...
[2017-07-13 10:29:45] [NOTICE] this node is the best candidate to be the new master, promoting...
[2017-07-13 10:29:46] [NOTICE] Redirecting logging output to '/var/log/repmgr/repmgr-9.6.log'
[2017-07-13 10:29:46] [ERROR] connection to database failed: could not connect to server: Connection refused
        Is the server running on host "server1" (172.134.11.45) and accepting
        TCP/IP connections on port 5432?

[2017-07-13 10:29:46] [ERROR] connection to database failed: could not connect to server: Connection refused
        Is the server running on host "PCYYYPFE" (172.134.11.99) and accepting
        TCP/IP connections on port 5432?

[2017-07-13 10:29:46] [NOTICE] promoting standby
[2017-07-13 10:29:46] [NOTICE] promoting server using '/usr/pgsql-9.6/bin/pg_ctl -D /appli/postgres/test_repmgr/9.6/data promote'
[2017-07-13 10:29:48] [NOTICE] STANDBY PROMOTE successful

~~~~

<!-- SAS&nbsp;: Comment sont-elles mesurées ? Et à quoi correspondent-elles ? -->
Regardons ensuite le résultat des insertions effectuées grâce à notre outil, comportant la chaîne de connexion ainsi qu'une résolution temporelle de 10ms (1 INSERT toute les 10ms).
Voici les mesures des insertions dans notre table de log lors de notre bascule vers notre nouveau primaire. (extrait)

|pk    | time                          |
|:----:|:-----------------------------:|
| 5876 | 2017-07-13 10:29:24.221169+02 |
| 5877 | 2017-07-13 10:29:24.277367+02 |
| 5878 | 2017-07-13 10:29:24.330156+02 |
| 5879 | 2017-07-13 10:29:24.388040+02 |
| 5880 | 2017-07-13 10:29:24.441046+02 |
| 5881 | 2017-07-13 10:29:24.493518+02 |
| 5882 | 2017-07-13 10:29:24.545547+02 |
| 5883 | 2017-07-13 10:29:24.597415+02 |
| 5884 | 2017-07-13 10:29:24.649334+02 |
| 5885 | 2017-07-13 10:29:24.701244+02 |
| 5886 | 2017-07-13 10:29:24.753549+02 |
| 5911 | 2017-07-13 10:29:46.685232+02 |
| 5912 | 2017-07-13 10:29:46.832526+02 |
| 5913 | 2017-07-13 10:29:46.886040+02 |
| 5914 | 2017-07-13 10:29:46.939793+02 |
| 5915 | 2017-07-13 10:29:46.997917+02 |
| 5916 | 2017-07-13 10:29:47.053968+02 |
| 5917 | 2017-07-13 10:29:47.108371+02 |

Les écritures ont repris au bout de 21 secondes.
Le failover client fonctionne donc comme attendu.


## Et ils vécurent heureux et eurent beaucoup de SELECT...

Nous avons de nouvelles écritures enregistrées au bout de 21 secondes, mais 25 INSERTS sont manquants (5886-5911).
Le client reçoit une erreur lorsque les insertions n'ont pas pu être faites.
Lorsque les écritures reprennent sur le nouveau primaire, le client nous informe des nouvelles insertions.
<!-- SAS&nbsp;: Comment voit-on les nouvelles écritures ? Que doit-on faire de ces inserts manquants ? Où
sont-ils ? Le client est-il prévenu ? -->
<!--LAV : idem SAS, je n'ai pas compris.-->


## Conclusion

<!-- SAS&nbsp;: PostgreSQL est génial. Mais je ne sais pas ce qui fonctionne, parce qu'on a perdu des
écritures. -->
La bascule **fonctionne** puisque nous retrouvons nos écritures qui continuent sur le nouveau primaire après la promotion.
Au final, l'incident n'a duré que 21 secondes et le failover a été inférieur à 1 seconde (client et base de données).

Nous avons pu constater que les écritures ont été mise en attente lorsque aucun nœud n'était disponible.
L'erreur remontée au client n'est pas produite ici, cependant elle permet d'avoir un avertissement que nous pouvons confirmer lors de l'analyse des traces et de prendre les dispositions nécessaires.
<!--LAV : que veux-tu dire par "pas documentée" ? -->

De plus, nous avons vu ensemble qu'une fois la bascule effectuée, il nous manquait des lignes dans notre table de log.
Il faut rejouer les insertions pour lesquelles le client a reçu une erreur.

<!--Il est possible d'éviter de perdre ces lignes, et aussi de les récupérer si tout se passe correctement.
En fonction de la configuration et de la solution de bascule automatique, je devrais, si nécessaire, restaurer mes données pour les réintégrer à ma nouvelle timeline.-->
<!--LAV: Il n'y a pas de perte d'écriture. Il y a eu un incident et le client a
reçu un message d'erreur lui disant que ses écritures n'ont pas été effectuées.
L'incident n'a duré que 21 secondes.-->

<!--Mais encore une fois il est tout à fait possible d'éviter cet écueil en modifiant la configuration. 
Dans le cas d'une réplication synchrone cependant, il faut garder à l'esprit qu'une panne sur le réplica entraine le bloquage de la primaire qui ne peut commiter les information sur l'intégralité du cluster.
Choix cornélien, il faut bien l'avouer. -->
<!--LAV : C'est pour cette raiuson que la communauté conseille de mettre 2 standbys
en réplication synchrone (avec un quorum de 1)-->


<!--Nous n'avons perdu "que" 25 écritures, soit 250 millisecondes (`10000µs*25`), malgré les 20 secondes d'indisponibilité.
C'est un moindre mal, mais si nous voulons garantir l'intégrité des données, il faut pouvoir mettre en oeuvre les moyens de conserver dans la mesure du possible cette intégrité.-->

<!-- Le temps pendant lequel aucune instance n'était disponible, les écritures ont été mises en attente par le client.-->

<!-- SAS&nbsp;: On les a perdu ou pas ? -->
<!--LAV : idem SAS-->


******
# note
Il est possible d'augmenter la fréquence d'échantillonnage de notre mesure. Cependant, j'ai choisi cette mesure pour tenter de respecter un ordre de grandeur déterminé par les échelles de mon système&nbsp;:

1. en dessous (largement) du temps total d'indisponibilité attendu (environ 20 secondes voir plus haut)
2. en dessous du temps nécessaire à la bascule une fois l'opération de basculement déclenchée (de l'ordre de 1s)
3. au dessus des temps machines&nbsp;: connexion à l'instance, logs système et PG, fréquence cpu... dans le cas contraire ma mesure aurait pu influencer le résultat par un trop grand nombre d'INSERTs à réaliser.

Ce qui aurait impacté les performances du serveur et de l'instance (imaginez une mesure toute les nanosecondes avec nanosleep&nbsp;: on atteindrait alors le milliard de tps&nbsp;!).
Avec une mesure toute les 10ms, nous avons alors 100 tps seulement pour le traçage des écritures.
Ce qui en soit correspond à un système avec une charge tout à fait honorable (environ 8 millions d'écritures par jour).
