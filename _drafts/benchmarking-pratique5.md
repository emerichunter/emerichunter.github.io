+++
title = "Benchmarking pratique (partie 5) - Pour aller plus loin"
date = "2017-06-02T09:30:11+01:00"
shortstory = "Vous souhaitez améliorer votre configuration pour votre instance Postgres ? Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer ? Hélas, vous ne pouvez pas prévoir leur impact à l'avance. Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données ? Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."
author = "Emeric Tabakhoff"
description = "Vous souhaitez améliorer votre configuration pour votre instance Postgres ? Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer ? Hélas, vous ne pouvez pas prévoir leur impact à l'avance. Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données ? Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."
banner = "banners/elephant-wrestling.jpg"
categories = ["technique"]
tags = ["automatisation"]
draft = true
+++


<!-- image 820 x 360 -->

Lors du PGDAY Paris de mars de cette année, j'ai pu assister à la conférence de Fabien Coelho sur le benchmarking que je vous ai retranscrit [ici](/post/proper-benchmarking/). 
J'ai aussi eu la chance de voir Kaarel Mopel de Cybertec mentionner pgbench-tools que je me suis empressé de regarder.



****
**couper ici pour la 3e partie**
le rapport rates_report n'est pas encore finalisé

****

## Régime "Normal" : la révolution dans le monde du bench

Nous allons donc choisir parmi les sets les plus intéressants les paramètres que nous allons tester


* les différents sous-régimes avec **SETRATES** : 10 100 250 500 et Latence limite 15ms.

Dans cette simulation, on tente de mettre en place une utilisation dite normale de la base de données. 

  * 10 tps : une base avec peu de trafic 
  * 100 tps : un trafic de production (8 millions de transactions par jour)
  * 250 tps : trafic de production important (20 millions de transactions journalières)
  * 500 tps : environnement avec un fort trafic 
  
 Je ne suis pas allé au-delà de ces valeurs, car passé cette limite, on entre dans un domaine peu courant qui ne fait plus partie d'une utilisation "moyenne".
 En effet, les tps maximum mesurées ici sont de l'ordre de 1500, c'est donc plus un mode de traitement batch.
 
 L'option latence limite permet de "skipper" toutes les transactions dont la latence est supérieure à une certaine valeur.
 Le but est de lisser le comportement pour obtenir une valeur moyenne. 
 C'est une option incluse par mes soins. 
 
 J'ai ajouté un nouveau rapport également pour observer la latence en fonction de taux de tps cible `rates_report`. 
 Le but est donc ici d'améliorer la latence en fixant les tps. 
		
* Limiter le nombre de transaction avec **TOTTRANS**


# Problèmes & contournements 


****

**partie suivante**

Le script python `log-to-csv` n'est pas compatible avec l'option **SETRATES** ce qui entraîne une absence de latence dans les logs pour 2 raisons : 
une colonne supplémentaire dans le fichier pgbench.log et la mention "skipped" pour les requêtes dépassant la valeur limite de latence.
J'ai donc ajouté un un nouveau script pour gérer ce cas `log-to-csv_rates` qui est appelé uniquement quand le paramètres SETRATES n'est pas laissé vide.

****

# Conclusion et discussion



****
**partie suivante**

* Retour sur la mémoire (buffers) : statistiques du pg_stat_bgwriter et répartition des écritures.

* Observation sur les checkpoints (fréquence, pourquoi, leur role et ce qu'il faut chercher à faire)

**Autre article buffers & checkpoints ?**



**brouillon à refaire**
D'autres fonctionnalités valent aussi la peine qu'on s'attarde dessus : log-to-csv, csv2gnuplot, dirty-plot, summary.




Le temps pour ce genre d'entreprise est très chronophage : un bench long (30 minutes) sérialisé de cette façon comptabilise 48h environ.
4 échelles, 6 clients, 4 taux cibles, 30 minutes de test, c'est 2880minutes (donc 48h).

**fin partie suivante**
****



# AMELIORATIONS FUTURES

* Fonctionnalités avancées :
log-to-csv, csv2gnuplot, dirty-plot, summary.
bufreport.sql, bufstats.sql, bufsummary.sql, fastest.sql, report.sql, summary.sql, tps-per-client.sql et les nouveaux : lowest_latency et fastest_w_low_lat

* remplacer pg_archiveshredder par un code propre qui retire des wals les transactions lors de l'initialisation de pgbench (avec une option)

* grapher la latence (90%< vs scale et clients) => OK

* extraire les tests les plus rapides dans limited_webreport dans un tableau 

* cumuler les paramètres améliorants les perfs un a un et les comparer sur un graphique final
  puis refaire le test en mode normal

* Wrapper pour faire des séries de test complètes ?

* Après l'intégration continue, le bench continu ?

* Reconfiguration "on the fly" 

* Auto Reconfiguration ?

* Obstacles ?

* Les tests en écriture séparés des tests en lecture : le cas de la réplication.

* Pourquoi le bench est une approximation : toutes les Bases sont uniques ! CONTOURNEMENT => Bench avec proxy (dédoubler les requêtes sur les serveurs

* Test sur lru et autovacuum - 3e partie ?

* FILLFACTOR = Ajouter au code puisqu'il est dans pgbench - OK

The secret art of tuning databases

bgwriter_delay = 100ms
lru_max
lru_multiplier 2,3 ,4, 5

cat /proc/sys/vm/overcommit_memory
cat /proc/sys/vm/overcommit_ratio
cat /proc/sys/vm/overcommit_kbytes

Kernel tweaks (EDB)
vm.dirty_background_ratio
vm.dirty_ratio
vm.shmmax
swappiness
shmall
overcommit_memory
kernel.transparent_hugepage.e
kernel.mmm.transparent_hugepage.d

	
# change the statistics :
random_page_cost
cpu_page_cost

* TEST tuning kernel linux complementaire
 I/O scheduler 
 	noop cft anticipatory decline 
 /sys/block/sd*/device/queue_depth
 64, 128, 256, 512, 1024
# cat /sys/block/sda/queue/nr_requests
128,256, 512, 1024, 8192
# echo 100000 > /sys/block/sda/queue/nr_requests


# Vers l'infini et au dela - 4e partie 

* Wrapper pour faire des séries de test complètes ?

* Après l'intégration continue, le bench continu ?

* Reconfiguration "on the fly" 

* Auto Reconfiguration ?

* Obstacles ?

* Les tests en écriture séparés des tests en lecture : le cas de la réplication. Et REPMGR ? 

* Pourquoi le bench est une approximation : toutes les Bases sont uniques !

* Test sur lru et autovacuum - 3e partie ?

* FILLFACTOR et intégration à pgbenchtools- 3e partie

* Réplication : répartition de charge
* lru 
* Autovacuum
* fill factor
* retour du checkpoint_timeout
* scheduler 
* nr_request
* dirty_ratio
* dirty_background_ratio

# L'AUTEUR (G. Smith)
