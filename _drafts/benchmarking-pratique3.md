+++
title = "Benchmarking pratique (partie 3)"
date = "2017-11-24T09:30:11+01:00"
shortstory = "Troisième partie&nbsp;: Pour comparer. Aujourd'hui, abordons un cas pratique. La méthode pour trouver les paramètres et les valeurs associées qui ont le plus d'impact sur votre configuration. Vous souhaitez améliorer votre configuration pour votre instance Postgres&nbsp;? Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer&nbsp;? Hélas, vous ne pouvez pas prévoir leur impact à l'avance. Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données&nbsp;? Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."
author = "Emeric Tabakhoff"
description = "Vous souhaitez améliorer votre configuration pour votre instance Postgres&nbsp;? Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer&nbsp;? Hélas, vous ne pouvez pas prévoir leur impact à l'avance. Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données&nbsp;? Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."
banner = "head/elephant-running.jpg"
categories = ["technique"]
tags = ["administration","performances"]
draft = true
+++


Lors des précédents chapîtres&nbsp;: [1](/post/benchmarking-pratique/) et [2](/post/benchmarking-pratique2/), je vous ai parlé de [pgbench-tools](https://github.com/gregs1104/pgbench-tools) de Greg Smith.
Nous avons ensemble fait le tour des fonctionnalités incluses dans l'outil.
Je vous propose aujourd'hui de passer à un cas pratique. 
Nous allons partir d'un contexte avec des contraintes imposées et d'autres plus flexibles.
Si nous n'avez pas installé l'outil, je vous conseille de le faire dès maintenant et pour l'installation vous pouvez vous référer à la partie précédente.


# Contexte

Me voici fraichement arrivé chez un nouveau client.
Je commence à prendre connaissance de l'existant.
Pour me familiariser avec l'outil de déploiement maison, on me demande de faire du bench.
Parfait timing, j'ai le temps et une idée derrière la tête.

Dans l'outil utilisé, une partie de l'utilitaire se trouve être en charge d'adapter la configuration de départ d'une instance Postgres en fonction des ressources materielles disponibles détectées.
Pourquoi ne pas regarder si les options choisies sont pertinentes&nbsp;? 
Faisons le tour des paramètres choisis.

* **shared_buffers**&nbsp;: correspond à 1/4 de la RAM présente sur le serveur - ne tient pas compte de la présence éventuelle d'autres instances ou d'application(s) (web par exemple)&nbsp;;
* **checkpoint_completion_target**&nbsp;: mis systématiquement à 0.9&nbsp;;
* **work_mem**&nbsp;: 1/50 de la RAM&nbsp;;
* **maintenance_work_mem**&nbsp;: 1/20 de la RAM&nbsp;;
* **checkpoint_timeout**&nbsp;: valeur par défaut 300s (5 minutes)&nbsp;;
* **effective_cache_size**&nbsp;: 3/4 de la RAM&nbsp;;
* **fill_factor**&nbsp;: n'est pas modifié (100 par défaut). Si je le mentionne, c'est qu'il va nous être utile.

Certains paramètres ne seront pas testés ici car le contexte ne s'applique pas ou le temps nous manque.
Tout d'abord le **SSL** restera en place car il est activé avec des certificats en place sur toute machine de PRODUCTION (voir le sujet traité par Fabien Cohelo [ici](/post/proper-benchmarking/)).
Aucun gain n'est donc possible de ce côté.
J'ajouterai même que la sécurité et l'intégrité viennent avant la performance (dans ce contexte).


* Le paramètre **synchronous_commit** reste enclenché (ON) car nous cherchons avant tout à privilégier l'intégrité des données&nbsp;;
* On active également **archive** (ON), comme expliqué dans mon [précédent post](/post/benchmarking-pratique/), on se met en situation réelle ce qui veut dire activation et archivage des WAL&nbsp;;
* **max_parallel_workers_per_gather** =2&nbsp;: ce paramètre dépendant essentiellement du nombre de vCPU/core disponible pour l'instance, il n'est pas nécessaire de le modifier car il est correct&nbsp;;
* **max_stack_depth**&nbsp;: stack_size du kernel moins une marge de sécurité d'1Mo pour éviter le crash de l'instance. C'est ici encore, bien calculé&nbsp;;
* **wal_buffers**&nbsp;: indexé sur le shared_buffers (1/32) par défaut. Cette valeur est une bonne valeur de départ. On conserve donc ce paramètre pour ne pas sombrer dans l'exhaustivité&nbsp;;
Le niveau de log pour debugging&nbsp;: il est conseillé de le garder à un niveau suffisant en production pour tracer l'activité sans impacter notablement les performances (checkpoints, connexions, déconnexions et requêtes supérieures à 500ms).

Il existe de nombreux autres  paramètres pouvant permettre une amélioration significative des performances. Cependant, il s'agit ici d'une entrée de blog et l'espace est donc limité.


# LA MACHINE

Nous sommes sur une machine virtuelle dont les ressources sont les suivantes&nbsp;:

* CentOS 6.6&nbsp;;
* PostgreSQL 9.6.2&nbsp;;
* 2 vCPU&nbsp;;
* 8Go RAM.
	
**Stockage**

 * 20Go Données&nbsp;;
 * 20Go Sauvegardes&nbsp;;
 * 10Go WAL.

Il s'agit d'une VM de test.
Les systèmes de fichiers sont tous sur le même Volume Physique (même volume groupe).
Cette machine va servir à tester pgbench-tools et illustrer ce qui a été déjà exposé en première et deuxième partie.


# LES TESTS

Comme expliqué précedemment, nous allons conduire deux grandes séries de tests.
Le régime en pic ou "Heure de pointe" permet de faire de petites pointes de vitesses qui vont nous servir à trouver une configuration de départ optimale en partant de la configuration initiale donnée par l'auto-configuration.
Dans un second temps, nous allons conduire une autre série de tests moins exhaustive sur les paramètres de notre instances, mais qui va d'avantage servir à déterminer la robustesse de cette configuration (sur une plus longue période de temps). 
Il conviendra de comparer pour finir la configuration initiale dans cette même situation pour pouvoir choisir la meilleure.

J'ai choisis de montrer uniquement sur les graphiques les jeux de données permettant de comparer un paramètre à la fois. 
Notre configuration de base et les nouvelles valeurs contre lesquelles le bench va nous permettre d'observer ou non un changement.
On appelle ceci une mesure différentielle.
Je vais vous montrer les graphiques et les tableaux choisis avec les valeurs qui vont nous permettre de converger vers une conclusion irréfutable (dans la mesure du possible).

## Protocole de test

Le processus se fait en 4 parties&nbsp;: 

1. Les tests de 1 minutes en régime heure de pointe sont comparés avec le paramètre à faire varier entre plusieurs valeurs&nbsp;;
2. A partir de ses données on tente de trouver la/les configurations en dégageant une tendance vers la latence la plus faible et les tps les plus élevées&nbsp;;
3. La ou les tests restant seront ensuite passé par le régime normal qui est plus long&nbsp;;
4. Pour conclure nous verrons si les performances se maintiennent pendant le régime normal (avec une cible pour le taux de tps) pour constater ou non une modification de la latence au cours du temps
(tout en continuant de monitorer les autres paramètres pour complémenter notre analyse).

**Dans le présent post, nous allons couvrir uniquement les points 1 et 2.** 
Les points suivants feront l'objet d'un autre post.

## Filtre d'amour

En raison du grand nombre de test et de set que j'ai personnellement effectué (environ 50 sets soit 3600 tests) cela se traduit par 1200 courbes.
Observer l'ensemble des courbes nuisant à la lisibilité, j'ai donc ajouté `limited_webreport` qui prend en entrée une série de sets séparés par des virgules.

		./limited_webreport 9,12,14


## Régime "Heure de pointe"

### configuration initial PG 

Comme expliqué plus haut, nous avons une configuration de départ sur notre instance.
	
 * **shared_buffers**&nbsp;: 1Go (c'est ici une erreur de l'autoconfiguration&nbsp;: **nous attendons 2Go**)&nbsp;;
 * **checkpoint_completion_target**&nbsp;: 0.9&nbsp;;
 * **work_mem**&nbsp;: 50Mo (**nous attendions ici 150Mo**), j'ai conservé la valeur par défaut pour la suite des tests&nbsp;;
 * **maintenance_work_mem**&nbsp;: 128Mo (**nous attendions 2,5Go**), j'ai également conservé cette valeur. Aucun n'a été effectué bench sur cette valeur&nbsp;;
 * **checkpoint_timeout**&nbsp;: 300s (5 minutes)&nbsp;;
 * **effective_cache_size**&nbsp;: 1536Mo (erreur du fichier de configuration original&nbsp;: **valeur  attendue 6Go**)&nbsp;;
 * **fillfactor**&nbsp;: 100.

Nous allons maintenant observer les résultats pour la modification du paramètre mémoire : `shared buffers`.
Les autres paramètres seront configurés aux valeurs attendues par l'autoconfiguration.

### shared buffers

 La valeur initiale était de 2Go, je décide de la multiplier par 2 et de la diviser par 2 alternativement.
 Sur les courbes 4Go&nbsp;: bleu; 2Go&nbsp;: rouge, 1Go&nbsp;: vert.
 


<table style="border: 0px;">
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/shard_buff_sets_sc_fac.png" alt="shared_buffers&nbsp;: tps/facteur d'échelle " style="width: 500px; border: 0px;"/></td>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/shard_buff_sets_clients.png" alt="shared_buffers&nbsp;: tps/nombre de clients" style="width: 500px; border: 0px;"/></td>  
  </tr>
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/shard_buff_sets_sc_fac_lat.png" alt="shared_buffers&nbsp;: latence/facteur d'échelle" style="width: 500px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/shard_buff_sets_clients_lat.png" alt="shared_buffers&nbsp;: latence/nombre de clients" style="width: 500px; border: 0px;"/></td>
  </tr>
</table>


Voici également le détail des valeurs collectées pendant les tests sous forme de tableau.


<table style="border: 0px;">
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_shard_buff_set22.png" alt="shared_buffers 2Go&nbsp;: set 22 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_shard_buff_set29.png" alt="shared_buffers 1Go&nbsp;: set 29 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_shard_buff_set30.png" alt="shared_buffers 4G&nbsp;: set 30 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>
  </tr>
 </table>




## Trouver la configuiration "optimale"

Armons-nous de notre méthodologie et de notre sens critique...

### 1. Tri par observation <!-- Comparaison visuelle -->

Cette comparaison est visuelle et nous allons utiliser nos yeux pour déterminer ici les sets les plus performants.
On compare les courbes de tps ainsi et de la latence ainsi que la façon dont celles-ci evoluent avec les paramètres de l'instance qui change.
On cherche donc une valeur de tps maximale (courbe supérieure) et une valeur de latence minimale (courbe inférieure).
Pour les tps, c'est assez facile, il suffit de regarder les courbes qui nous donnent un score absolu.

Pour la latence, que choisir&nbsp;? Nous avons le choix pour les valeurs&nbsp;: AVG, 90e percentile, MAX&nbsp;?
Je vous ai expliqué ce qu'était le 9e décile (ou 90e percentile) dans le post précédent. 
La valeur MAX est une valeur unique qui n'a rien de statistique et ne peut en rien intervenir dans notre choix car il ne s'agit que d'une valeur parmi des milliers.
La valeur moyenne pourrait être notre premier choix mais tout d'abord faisons un peu de lecture&nbsp;: [ici](https://www.dynatrace.com/blog/why-averages-suck-and-percentiles-are-great/)) par exemple.

Nous allons finalement choisir le 90e percentile.


#### _**Légende**_&nbsp;:

 * \>&nbsp;: signifie que le membre de gauche est **supérieur**  que le membre de droite
 * \>>&nbsp;: signifie que le membre de gauche est **très supérieur** au membre de droite
 
 
 
**Shared_buffers**&nbsp;: 

 * _tps_&nbsp;: 2Go\>4Go\>1Go
 * _latence_&nbsp;: 2Go\<4Go\<\<1Go

On remarque que pour les petits facteurs d'échelle et pour un faible nombre de clients les courbes sont assez proches les unes des autres.
Dans le case de 2 et 4 Go, elles se confondent même pour finir par afficher un resultat largement en faveur des 2Go préconisés.
C'est très intéressant&nbsp;: l'allocation du cache se comporte parfaitement lorsque la taille de la base de données reste dans une configuration principalement en cache ou RAM (1, 10, 100).
Cependant, lorsque nous dépassons cette valeur, le résultat devient contre productif.
De même, l'utilisation du cache dépendant du nombre de connexions simultanées, on voit apparaître un "seuil de contre-productivité" passé 8 clients (latence et tps).
 
# Temps de bench total

Combien de temps a donc nécessité ce test ?
C'est une question centrale ici, car le bench est une activité chronophage et tous les moyens sont bons pour gagner du temps.

Chaque courbe nécessite **72 minutes** de test (voir [chapitre précedent](/post/benchmarking-pratique2#vacuum) pour ce calcul).
Le chargement (`INSERT`) et le nettoyage des tuples morts (`VACUUM`) n'est pas comptabilisé dans ce calcul.
Je peux, d'après mon expérience avec `pgbench-tools`, estimer ces opération autour de 10 à 15 minutes pour ces tests (ceci varie évidemment avec l'augmentation du facteur d'échelle).
En arrondissant à 90 minutes pour chaque test, on peut donc avoir une estimation finale de 270 minutes (4 heures et 30 minutes), pour tester 3 valeurs d'un seul paramètre.
 
# Problemes rencontrés, contournements et améliorations
<!--Pas indispensable-->
<!--GNUplot a été difficile à installer car c'est un ancien outil et qu'il n'était pas présent sur l'installation de base de mon système.
L'impossibilité de faire appel à yum ou apt car les serveurs sont isolés du monde et qu'il n'existe pas de mirroirs internes qui soient à l'image des repositories officiel a donc allongé cette partie.
Parfois, la sécurité peut nuire à l'agilité et c'était ici le cas.-->

Postgres-9.6 a changé le format des valeurs aléatoires.
Ce n'était pas une grosse difficulté, mais pour toute personne qui vient d'aborder le sujet et/ou qui ne connait pas la programmation, c'est un facteur limitant.
Cela fait un moment que le projet pgbench-tools n'a pas subit d'améliorations.
Mes pull-request sont pour le moment toujours en attentent après plus de 6 mois de proposition. 
J'ai donc décidé de forker le projet et de pousser des modifications et améliorations car je pense que c'est outil qui en vaut la peine.
Je vais tenter par la suite d'ajouter des fonctionnalités tout en conservant le plus possible une certaine ergonomie (surtout quand on fait des milliers de tests).
J'ai décidé d'explorer des voies différentes, vous en saurez plus au prochain chapître.

La gestion des comparatifs avec `./webreport` était loin de me convenir.
En effet, on a à chaque fin de test, la génération de tout le rapport pour tous les tests effectués.
L'impact et la durée n'ont pas d'importance, mais la lisibilité et l'exploitabilité des résultats est ici mise en cause.
Comment faire de l'analyse avec 25 ou 50 courbes différentes qui ont parfois la même couleur, parfois une échelle complètement différente...
C'est tout simplement impossible.
J'ai donc ajouté `limited_webreport` qui fait cela pour vous.
On a un beau rapport avec uniquement les données concernant les tests que l'on souhaite voir apparaitre.

De plus, j'ai ajouté `latest_set` qui permet de ne visualiser que la progression du dernier set.

L'autre ennui avec les tests en grand nombre, c'est qu'il y a génération de beaucoup d'erreur.
`cleanup` permet de supprimer tous les tests dont les tps sont à 0.
`cleanup_fromvalue` permet quant à lui de supprimer tous les tests supérieurs à une valeur entrée en paramètre (attention il n'y a pas de borne supérieure).
C'est assez utile quand vous vous rendez compte que vous avez lancé votre test après avoir modifié le paramètre dans le fichier de configuration sans redémarrer l'instance.
Vous pouvez alors tout supprimer sans manipulation hasardeuse.

La liste des tests issu du rapport `report` donné par la commande 

		psql -d results -f reports/report.sql
		
est ordonné pour suivre plus facilement l'évolution par ordre chronologique des tests avec `list_orderbytest`

Pgbench comporte l'option FILLFACTOR depuis la version 8.2 (**à confirmer**), il me semblait normal de l'ajouter à l'outil. 
Il ne prend ici qu'une seule valeur, mais c'est amplement suffisant à mon sens.

Si notre serveur se retrouve avec un système de fichier plein et que Postgres plante, si la collecte des informationsn système a été activée, les fichiers de logs `vmstat`, `iostat` et `meminfo` qui restent ouvert (`deleted`).
Ces fichiers continuerons de prendre de l'espace et risque de remplir la partition par exemple.
Pour retrouver les fichiers concernés, voici une commande qui peut vous aider :

~~~~
lsof | grep delete
~~~~

Meme une fois supprimés ils continuent de prendre de l'espace disque (inode toujours existant). 
On peut récupérer l'espace en rebootant la machine ou en employant `echo '' >` dans les fichiers si l'on se trouve dans un cas où on ne peut pas redémarrer immédiatement (_e.g._ en plein test).
Si le test est terminé, il est possible de faire un kill sur le process ce qui libèrera l'inode et la mémoire associée.


# CONCLUSION & DISCUSSION 


Il ne faut pas prendre les résultats des tests présentés ici comme parole d'évangile.
Je vous présente un outil, et une configuration de départ.
J'applique par la suite un **raisonnement critique** pour tenter de départager des séries de test.
Ce qui s'applique ici, peut ne pas s'appliquer pour vous.
De plus, la machine sur laquelle j'ai effectué ces tests semblait donner des résultats différents d'un jour à l'autre (en fonction des ressources disponibles pour la machine virtuelle).
C'est donc un argument supplémentaire pour justifier d'effectuer vos propres tests avec votre configuration.
L'important dans la démarche présentée ici est le **raisonnement**.

Once upon a time in the Bench...