+++
title = "Benchmarking pratique (partie 4)"
date = "2017-06-02T09:30:11+01:00"
shortstory = "Troisième partie&nbsp;: Pour comparer encore. Aujourd'hui, abordons un cas pratique. La méthode pour trouver les paramètres et les valeurs associées qui ont le plus d'impact sur votre configuration. Vous souhaitez améliorer votre configuration pour votre instance Postgres ? Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer ? Hélas, vous ne pouvez pas prévoir leur impact à l'avance. Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données ? Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."
author = "Emeric Tabakhoff"
description = "Vous souhaitez améliorer votre configuration pour votre instance Postgres ? Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer ? Hélas, vous ne pouvez pas prévoir leur impact à l'avance. Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données ? Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."
banner = "head/jumping-elephant.jpg"
categories = ["technique"]
tags = ["administration","performances"]
draft = true
+++

<!--
Lors des précédents chapîtres&nbsp;: [1](/post/benchmarking-pratique/) et [2](/post/benchmarking-pratique2/), je vous ai parlé de [pgbench-tools](https://github.com/gregs1104/pgbench-tools) de Greg Smith.
Nous avons ensemble fait le tour des foncionnalités incluses dans l'outil.
Je vous propose aujourd'hui de passer à un cas pratique.
Nous allons partir d'un contexte avec des contraintes imposées et d'autres plus flexibles.
Si nous n'avez pas installé l'outil, je vous conseille de le faire dès maintenant et pour l'installation vous pouvez vous référer à la partie précédente.
-->

Lors des précédents chapîtres&nbsp;: [1](/post/benchmarking-pratique/) et [2](/post/benchmarking-pratique2/), je vous ai parlé de [pgbench-tools](https://github.com/gregs1104/pgbench-tools) de Greg Smith.
Aujourd'hui je vous propose de continuer dans la présentation des tests de bench comme dans le volet précédent [3](/post/benchmarking-pratique3/).
Cette fois, nous allons observer les effets de `checkpoint_timeout`, `checkpoint_completion_target` et `FILLFACTOR`.
Je vous présenterai ensuite une série de scripts permettant de trouver les sets/tests les plus rapides avec une grande quantité de données (nombreux sets).

## Suite des tests

### Rappel de la configuration de départ 

Comme expliqué plus haut, nous avons une configuration de départ sur notre instance.

 * shared_buffers&nbsp;: 1Go (c'est ici une erreur de l'autoconfiguration&nbsp;: **nous attendons 2Go**)&nbsp;;
 * **checkpoint_completion_target**&nbsp;: 0.9&nbsp;;
 * work_mem&nbsp;: 50Mo (nous attendions ici 150Mo), j'ai conservé la valeur par défaut pour la suite des tests&nbsp;;
 * maintenance_work_mem&nbsp;: 128Mo (nous attendions 2,5Go), j'ai également conservé cette valeur. Aucun n'a été effectué bench sur cette valeur&nbsp;;
 * **checkpoint_timeout**&nbsp;: 300s (5 minutes)&nbsp;;
 * effective_cache_size&nbsp;: 1536Mo (erreur du fichier de configuration original&nbsp;: **valeur  attendue 6Go**)&nbsp;;
 * **fillfactor**&nbsp;: 100.

### checkpoint_timeout

Sur ce test, on fait varier la valeur de `checkpoint_timeout`.
La valeur par défaut est 300s, on compare avec 600s.
J'ai lu à plusieurs reprises et notemment par Greg Smith que cette valeur par défaut est obsolète et **trop basse**.
Certains experts conseillent souvent 600.
La façon de procéder pour trouver la bonne valeur pour une instance consiste à augmenter cette valeur jusqu'à diminution (voire disparition) de `checkpoint_req` dans la vue `pg_stat_bgwriter`.
**Attention**&nbsp;: Comme spécifié dans la documentation, l'augmentation de ce paramètre influe également sur la durée possible de de restauration de votre instance lors d'un crash.
Il faut donc peser le pour et le contre lors de la modification de ce paramètre.


<table style="border: 0px;">
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/checkpoint-tmout-scaling-sets.png" alt="checkpoints_timeout&nbsp;: tps/facteur d'échelle " style="width: 500px; border: 0px;"/></td>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/checkpoint-tmout-clients-sets.png" alt="checkpoints_timeout&nbsp;: tps/nombre de clients" style="width: 500px; border: 0px;"/></td>  
  </tr>
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/checkpoint-tmout-scaling-latency-sets.png" alt="checkpoints_timeout&nbsp;: latence/facteur d'échelle" style="width: 500px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/checkpoint-tmout-clients-latency-sets.png" alt="checkpoints_timeout&nbsp;: latence/nombre de clients" style="width: 500px; border: 0px;"/></td>
  </tr>
</table>

Comme dans les précédents tests, voici également les tableaux avec les valeurs de latence détaillées.

<table style="border:0px;">
  <tr style="border:0px;">
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_chkpt_tmout_set14.png" alt="checkpoint_timeout 600&nbsp;: set 14 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>
	<td style="border:0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_effective_c_sz_set22.png" alt="checkpoint_timeout 300&nbsp;: set 22 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>
    <td style="border:0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_checkpoint_timeout_set40.png" alt="checkpoint_timeout 1200&nbsp;: set 40 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>
  </tr>
</table>


### checkpoint_completion_target


La valeur de configuration par défaut de PostgreSQL étant 0.5, on conseille souvent de passer la valeur 0.9 dans ce paramètre.
Celui-ci permet de compléter d'avantage de wal_segment avant un flush sur disque.
J'ai voulu voir si en baissant un peu ce paramètre nous pouvions voir un effet direct (0.8 et 0.9).
J'ai également fait un test avec 2 valeurs différentes pour checkpoint_timeout (300 et 600) pour mettre en perspective.

<!--
**NdA**&nbsp;: 
*  plutot tester 3 valeurs (0.7, 0.8, 0.9), ou (0.8, 0.9, 0.95) ?
-->


<table style="border: 0px;">
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/chkpt_target_sets_sc_factor.png" alt="checkpoints_completion_target 0.8 (rouge et violet) et 0.9 (bleu et vert)&nbsp;: tps/facteur d'échelle " style="width: 500px; border: 0px;"/></td>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/chkpt_target_sets_clients.png" alt="checkpoints_completion_target 0.8 (rouge et violet) et 0.9 (bleu et vert)&nbsp;: tps/nombre de clients" style="width: 500px; border: 0px;"/></td>  
  </tr>
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/chkpt_target_sets_sc_fac_lat.png" alt="checkpoints_completion_target 0.8 (rouge et violet) et 0.9 (bleu et vert) &nbsp;: latence/facteur d'échelle" style="width: 500px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/chkpt_target_sets_clients_lat.png" alt="checkpoints_completion_target 0.8 (rouge et violet) et 0.9 (bleu et vert) &nbsp;: latence/nombre de clients" style="width: 500px; border: 0px;"/></td>
  </tr>
</table>


<table style="border: 0px;">
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_shard_buff_set22.png" alt="effective_cache_size 6Go, checkpoint_timeout 300, checkpoints_completion_target 0.9&nbsp;: set 22 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_chkpt_tgt_set24.png" alt="effective_cache_size 6Go, checkpoint_timeout 300, checkpoints_completion_target 0.8&nbsp;: set 24 par facteur d'échelle et clients" style="width: 350px; border: 0px;"/></td>  
  </tr>
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_chkpt_tgt_set13.png" alt="effective_cache_size 6Go, checkpoint_timeout 600, checkpoints_completion_target 0.8&nbsp;: set 13 par facteur d'échelle et clients" style="width: 350px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_chkpt_tgt_set14.png" alt="effective_cache_size 6Go, checkpoint_timeout 600, checkpoints_completion_target 0.9&nbsp;: set 14 par facteur d'échelle et clients" style="width: 350px; border: 0px;"/></td>
  </tr>
</table> 


### FILLFACTOR

Voici ici le dernier paramètre que nous faisons varier pour cette série de test. 
Pour une meilleure compréhension, je vous conseille pour débuter de lire la [documentation](https://www.postgresql.org/docs/current/static/sql-createtable.html).
Pour simplifier ce paramètre a une influence sur les performances des UPDATEs.
La valeur initiale est ici 100.
Je teste 95 et 90 en plus.


<table style="border: 0px;">
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/fillfactor_sets_sc_fac.png" alt="fillfactor 100 rouge, 95 bleu, 90 vert&nbsp;: tps/facteur d'échelle " style="width: 500px; border: 0px;"/></td>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/fillfactor_sets_clients.png" alt="fillfactor 100 rouge, 95 bleu, 90 vert&nbsp;: tps/nombre de clients" style="width: 500px; border: 0px;"/></td>  
  </tr>
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/fillfactor_sets_sc_fac_lat.png" alt="fillfactor 100 rouge, 95 bleu, 90 vert &nbsp;: latence/facteur d'échelle" style="width: 500px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/fillfactor_sets_clients_lat.png" alt="fillfactor 100 rouge, 95 bleu, 90 vert &nbsp;: latence/nombre de clients" style="width: 500px; border: 0px;"/></td>
  </tr>
</table>

Le détail des sets dans les tableaux.

<table style="border: 0px;">
  <tr>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_shard_buff_set22.png" alt="fillfactor 100&nbsp;: set 22 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>  
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_fillfactor_set32.png" alt="fillfarctor 95&nbsp;: set 32 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>
    <td style="border: 0px;"> <img src="/images/post/pgbench-tools-2017/heuredepointe/tableau_fillfactor_set33.png" alt="fillfactor 90&nbsp;: set 33 par facteur d'échelle et clients" style="width: 250px; border: 0px;"/></td>
  </tr>
</table>


## Trouver la configuiration "optimale"

### 1. Tri par observation <!--Comparaison visuelle-->

Cette comparaison est empirique et nous allons utiliser nos yeux pour déterminer ici les sets les plus performants.
On compare les courbes de tps ainsi et de la latence ainsi que la façon dont celles-ci evoluent avec les paramètres de l'instance qui change.
On cherche donc une valeur de tps maximale (courbe supérieure) et une valeur de latence minimale (courbe inférieure).
Pour les tps, c'est assez facile, il suffit de regarder les courbes qui nous donnent un score absolu.

Pour la latence, que choisir ? Nous avons le choix pour les valeurs&nbsp;: AVG, 90e percentile, MAX ?
Je vous ai expliqué ce qu'était le 9e décile (ou 90e percentile) dans le post précédent. 
La valeur MAX est une valeur unique qui n'a rien de statistique et ne peut en rien intervenir dans notre choix car il ne s'agit que d'une valeur parmi des milliers.
La valeur moyenne pourrait être notre premier choix mais tout d'abord faisons un peu de lecture&nbsp;: [ici](https://www.dynatrace.com/blog/why-averages-suck-and-percentiles-are-great/)) par exemple.

Nous allons finalement choisir le 90e percentile.


#### _**Légende**_&nbsp;:

 * \>&nbsp;: signifie que le membre de gauche est plus grand  que le membre de droite 
 * \>>&nbsp;: signifie que le membre de gauche est très supérieur au membre de droite
 

**Checkpoint_timeout**&nbsp;: 

C'est pour moi quelque peu surprenant, mais il faut bien admettre qu'ici une valeur trop élevée impacte négativement les tps comme la latence.

 * tps&nbsp;: 600\>300\>1200
 * latence&nbsp;: 600\<300\<1200


Il est à noter que ce paramètre devrait être testé dans des durées de test supérieures à `checkpoint_timeout` donc supérieures à 10 minutes. 
Il convient donc de tester ces 2 valeurs en régime normal avec une durée suffisamment longue&nbsp;: 2 à 3 fois la valeur du paramètre.

Je pense que le manque de performance de 1200 peut etre lié au fait que nous ne faisons que des tests d'1 minutes.

Je propose donc de refaire un test plus long en conservant les 3 mêmes valeurs mais avec des durées d'une heure (ce qui fait respectivement 12, 6, et 3 checkpoints).
On peut alors faire redescendre la valeur `SETTIMES` de notre fichier `config` à 1 au lieu de 3. 
Nous aurions dans ce cas 3 sets à conduire faisant chacun&nbsp;:  
**6** valeurs de clients (1, 2, 4, 8, 16, 32) * **4** facteurs d'échelles (1, 10, 100, 1000) * 1h de test = **24** heures de bench pour chaque paramètre.

Je complémenterai donc avec ce test pour une prochaine fois.

**Checkpoint_completion_target**&nbsp;: 

Si le paramètre précédent était difficile à analyser, celui-ci en revanche est assez facile. 
Les résultats sont sans appel.
Nous devons comparer les courbes 2 à 2. 
Vert (0.9) et rouge (0.8) montrent un clair avantage à la courbe verte tps et latence. 
Bleu (0.9) et violet (0.8) nous montrent la même tendance avec cependant un écart relatif moins important.


**FILLFACTOR**&nbsp;: 

 * tps&nbsp;: 90\>95\>>100
 * latence&nbsp;: 90\<95\<\<100

Quand on voit un effet de cette importance, on est tenté de crier au victoire, cependant je tiens à mettre en garde le lecteur.
En effet, les scripts tpc-b comportent un grand nombre d'UPDATE. 
La structure des données ne comporte que 4 tables et FILLFACTOR est appliqué sur toutes les tables sans distinction.
Ceci fait de ce paramètre un outils qu'il ne faut surtout pas généraliser.

Effectivement, le gain de performance est significatif, mais il convient d'appliquer ce paramètre sur des tables qui en ont besoin et non sur l'ensemble de la base.
Je rappelle que les pages pré-allouées on un impact en terme d'espace disque qui est non négligeable.
Preuve en est que mon disque de données a saturé avec FILLFACTOR=90.
C'est pourquoi la courbe en fonction du facteur d'échelle s'arrete pour celle-ci à 900 au lieu de 1000.



### 2. Outils de traitement statistiques

Tentons maintenant une approche plus fine.

Avec un grand nombre de test, il devient difficile de tous les mettre sur un seul et même graphique car la lisibilité est trop limitée.
J'avais ajouté `limited_webreport` pour cette raison, mais cette fois si nous souhaitons prendre en compte tous les tests effectués pour trouver les plus performants, l'outil graphique trouve ses limites.
Nous passons donc par des outils de traitement permettant de dégager les meilleurs sets de façon plus méthodique et impartiale mais également sur la globalité des tests réalisés.

L'outils pgbench-tools vient avec un lot de rapports permettant de trier les tests. 
Tout d'abord les plus rapides (ici en tps), c'est à dire par ordre décroissant.

~~~~
psql -d results -f reports/fastest.sql
 set |    script    | scale | clients | workers | tps
-----+--------------+-------+---------+---------+------
  4 | tpc-b_96.sql |   100 |      32 |       2 | 2246
 33 | tpc-b_96.sql |   100 |      32 |       2 | 2209
  2 | tpc-b_96.sql |   100 |      32 |       2 | 2178
  6 | tpc-b_96.sql |   100 |      32 |       2 | 2170
  5 | tpc-b_96.sql |   100 |      32 |       2 | 2139
 18 | tpc-b_96.sql |   100 |      32 |       2 | 2021
 32 | tpc-b_96.sql |   100 |      32 |       2 | 1971
  4 | tpc-b_96.sql |   100 |      16 |       2 | 1962
 33 | tpc-b_96.sql |   900 |      32 |       2 | 1922
  5 | tpc-b_96.sql |    10 |      32 |       2 | 1902
  4 | tpc-b_96.sql |    10 |      32 |       2 | 1890
 32 | tpc-b_96.sql |   100 |      16 |       2 | 1874
 33 | tpc-b_96.sql |   100 |      16 |       2 | 1855
 33 | tpc-b_96.sql |    10 |      32 |       2 | 1849
  6 | tpc-b_96.sql |    10 |      32 |       2 | 1847
  2 | tpc-b_96.sql |    10 |      32 |       2 | 1844
 18 | tpc-b_96.sql |   100 |      16 |       2 | 1836
  4 | tpc-b_96.sql |    10 |      16 |       2 | 1835
 44 | tpc-b_96.sql |    10 |      32 |       2 | 1821
  2 | tpc-b_96.sql |   100 |      16 |       2 | 1818
(20 rows)
~~~~

J'ai ajouté des traitements qui permettent de faire le meme type de tri en cherchant les latences les plus basses.
~~~~
psql -d results -f reports/lowest_latency.sql
set |    script    | scale | clients | workers | p90_latency
-----+--------------+-------+---------+---------+-------------
 44 | tpc-b_96.sql |     1 |       1 |       1 |       1.865
 45 | tpc-b_96.sql |     1 |       1 |       1 |        1.87
 58 | tpc-b_96.sql |    10 |       1 |       1 |       1.879
 37 | tpc-b_96.sql |     1 |       1 |       1 |       1.885
 46 | tpc-b_96.sql |     1 |       1 |       1 |       1.886
 35 | tpc-b_96.sql |     1 |       1 |       1 |       1.899
 26 | tpc-b_96.sql |     1 |       1 |       1 |        1.91
 58 | tpc-b_96.sql |     1 |       1 |       1 |       1.912
 44 | tpc-b_96.sql |    10 |       1 |       1 |       1.912
 33 | tpc-b_96.sql |     1 |       1 |       1 |       1.914
 45 | tpc-b_96.sql |    10 |       1 |       1 |        1.92
 22 | tpc-b_96.sql |     1 |       1 |       1 |       1.924
 23 | tpc-b_96.sql |     1 |       1 |       1 |       1.937
 46 | tpc-b_96.sql |    10 |       1 |       1 |       1.937
 59 | tpc-b_96.sql |     1 |       1 |       1 |       1.937
 27 | tpc-b_96.sql |     1 |       1 |       1 |       1.937
 13 | tpc-b_96.sql |     1 |       1 |       1 |       1.937
 10 | tpc-b_96.sql |     1 |       1 |       1 |        1.94
 56 | tpc-b_96.sql |     1 |       1 |       1 |       1.941
 43 | tpc-b_96.sql |     1 |       1 |       1 |       1.944
(20 rows)
~~~~


Pour bien faire, j'ai continué avec un croisement les résultats&nbsp;: tps le plus élevé avec latence la plus basse.
Ici on commence à avoir une vision plus complète de l'ensemble de nos résultats. 
Cependant, le rapport présenté est trop global.


~~~~
psql -d results -f reports/fastest_w_lowest_lat.sql
set |    script    | scale | clients | workers | tps  | set |    script    | scale | clients | workers | p90_latency
-----+--------------+-------+---------+---------+------+-----+--------------+-------+---------+---------+-------------
 33 | tpc-b_96.sql |   100 |      32 |       2 | 2209 |  33 | tpc-b_96.sql |   100 |      32 |       2 |      21.902
 32 | tpc-b_96.sql |   100 |      32 |       2 | 1971 |  32 | tpc-b_96.sql |   100 |      32 |       2 |      24.719
 33 | tpc-b_96.sql |   900 |      32 |       2 | 1922 |  33 | tpc-b_96.sql |   900 |      32 |       2 |      25.234
 32 | tpc-b_96.sql |   100 |      16 |       2 | 1874 |  32 | tpc-b_96.sql |   100 |      16 |       2 |      12.877
 33 | tpc-b_96.sql |   100 |      16 |       2 | 1855 |  33 | tpc-b_96.sql |   100 |      16 |       2 |      12.668
 33 | tpc-b_96.sql |    10 |      32 |       2 | 1849 |  33 | tpc-b_96.sql |    10 |      32 |       2 |      33.168
 44 | tpc-b_96.sql |    10 |      32 |       2 | 1821 |  44 | tpc-b_96.sql |    10 |      32 |       2 |      33.814
 33 | tpc-b_96.sql |   900 |      16 |       2 | 1816 |  33 | tpc-b_96.sql |   900 |      16 |       2 |       13.29
  9 | tpc-b_96.sql |    10 |      16 |       2 | 1801 |   9 | tpc-b_96.sql |    10 |      16 |       2 |      15.461
 32 | tpc-b_96.sql |  1000 |      32 |       2 | 1801 |  32 | tpc-b_96.sql |  1000 |      32 |       2 |      27.642
  8 | tpc-b_96.sql |    10 |      32 |       2 | 1798 |   8 | tpc-b_96.sql |    10 |      32 |       2 |      34.357
 45 | tpc-b_96.sql |    10 |      32 |       2 | 1793 |  45 | tpc-b_96.sql |    10 |      32 |       2 |      34.504
 10 | tpc-b_96.sql |    10 |      32 |       2 | 1792 |  10 | tpc-b_96.sql |    10 |      32 |       2 |      34.338
 42 | tpc-b_96.sql |    10 |      32 |       2 | 1783 |  42 | tpc-b_96.sql |    10 |      32 |       2 |      34.575
  9 | tpc-b_96.sql |    10 |      32 |       2 | 1781 |   9 | tpc-b_96.sql |    10 |      32 |       2 |      34.494
  7 | tpc-b_96.sql |    10 |      32 |       2 | 1781 |   7 | tpc-b_96.sql |    10 |      32 |       2 |      34.502
 39 | tpc-b_96.sql |    10 |      32 |       2 | 1778 |  39 | tpc-b_96.sql |    10 |      32 |       2 |      34.606
  8 | tpc-b_96.sql |    10 |      16 |       2 | 1777 |   8 | tpc-b_96.sql |    10 |      16 |       2 |      15.751
 46 | tpc-b_96.sql |    10 |      32 |       2 | 1771 |  46 | tpc-b_96.sql |    10 |      32 |       2 |      35.212
 58 | tpc-b_96.sql |    10 |      16 |       2 | 1768 |  58 | tpc-b_96.sql |    10 |      16 |       2 |      15.871
(20 rows)

~~~~

Le binôme de ce script&nbsp;: la latence la plus faible avec le nombre de tps le plus élevé (`lowest_lat_w_fastest`).
Nous avons à chaque fois uniquement les 20 résultats les plus performants affichés.


Mais hélas, avec un grand nombre de test, il devient difficile de faire un comparatif sur un facteur d'échelle précis et/ou un nombre de clients en particulier.
De même lorsque l'on cherche la latence la plus basse, on tombe souvent éloigné des tps les plus élevées, et inversement. 
Alors comment trouver un compromis ?

#### Toujours faire des compromis...

L'étape finale consiste donc à trouver un compromis entre latence basse et tps élevé.
J'ai donc fait un dernier script sql qui prend des valeurs limites en paramètre&nbsp;: `compromise_params.sql`. 
Dans l'exemple ci dessous, les tps (\>=700) et la latence (=\<15) ainsi qu'un facteur d'échelle fixe (900-1000) et pour 1 à 16 clients.
Ceci nous donne une fenêtre plus étroite pour regarder l'ensemble des jeux de résultats dans ce qui peut commencer à ressembler (pour ma part) à un entrepôt de résultat.

~~~~
psql -d results -v lat=15 -v tps=700 -v lscale=900 -v hscale=1000 -v lclients=1 -v hclients=16 -f reports/compromise_params.sql
 set |                                                  info                                                  |    script    | scale | clients | workers | tps  | p90_latency
-----+--------------------------------------------------------------------------------------------------------+--------------+-------+---------+---------+------+-------------
  33 | Eff_csize 6G shrd_buff 2G FILLFAC_90 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b                 | tpc-b_96.sql |   900 |      16 |       2 | 1816 |       13.29
  32 | Eff_csize 6G shrd_buff 2G FILLFAC_95 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b                 | tpc-b_96.sql |  1000 |      16 |       2 | 1601 |      14.518
  33 | Eff_csize 6G shrd_buff 2G FILLFAC_90 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b                 | tpc-b_96.sql |   900 |       8 |       2 | 1257 |       8.756
  32 | Eff_csize 6G shrd_buff 2G FILLFAC_95 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b                 | tpc-b_96.sql |  1000 |       8 |       2 | 1228 |       9.393
  22 | Effective_csize 6G chkpt_tout 300 ckpt_completion_tgt=0.9  archiveon shared_buffers 2G tpc-b (3nd)     | tpc-b_96.sql |  1000 |       8 |       2 |  998 |      11.173
  23 | Effective_csize 4G chkpt_tout 600 ckpt_completion_tgt=0.9  archiveon shared_buffers 2G tpc-b           | tpc-b_96.sql |  1000 |       8 |       2 |  991 |      11.391
   8 | Effective_csize 1.5G chkpt_tout 600 archiveon tpc-b                                                    | tpc-b_96.sql |  1000 |       8 |       2 |  985 |      11.443
  44 | Eff_csize 6G shrd_buff 2G bgw_dl100_mul3_maxp512 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b     | tpc-b_96.sql |  1000 |       8 |       2 |  981 |      11.322
   9 | Effective_csize 1.5G chkpt_tout 600 archiveon shared_buffers 2G tpc-b                                  | tpc-b_96.sql |  1000 |       8 |       2 |  978 |      11.852
  11 | Effective_csize 1.5G chkpt_tout 600 workmem 100MB archiveon shared_buffers 2G tpc-b                    | tpc-b_96.sql |  1000 |       8 |       2 |  970 |      11.841
  14 | Effective_csize 6G chkpt_tout 600 ckpt_completion_tgt=0.9  archiveon shared_buffers 2G tpc-b           | tpc-b_96.sql |  1000 |       8 |       2 |  970 |      12.122
  43 | Eff_csize 6G shrd_buff 2G bgw_delay_100 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b              | tpc-b_96.sql |  1000 |       8 |       2 |  965 |      11.564
  46 | Eff_csize 6G shrd_buff 2G bgw_dl10_mul4_maxp512 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b      | tpc-b_96.sql |  1000 |       8 |       2 |  960 |       11.49
  49 | Eff_csize 6G shrd_buff 2G bgw_dl100_mul6_maxp512 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b     | tpc-b_96.sql |  1000 |       8 |       2 |  960 |      11.631
  55 | Eff_csize 6G shrd_buff 2G bgw_dl100_mul3_maxp512 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b (2) | tpc-b_96.sql |  1000 |       8 |       2 |  960 |      11.777
  10 | Effective_csize 1.5G chkpt_tout 600 workmem 25MB archiveon shared_buffers 2G tpc-b                     | tpc-b_96.sql |  1000 |       8 |       2 |  960 |      11.895
  33 | Eff_csize 6G shrd_buff 2G FILLFAC_90 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b                 | tpc-b_96.sql |   900 |       4 |       2 |  959 |       5.676
  41 | Eff_csz 6G chkpt_tout 1200 ckpt_c_tgt=0.8  archiveon shared_buffers 2G tpc-b                           | tpc-b_96.sql |  1000 |       8 |       2 |  956 |      11.843
  45 | Eff_csize 6G shrd_buff 2G bgw_dl100_mul4_maxp512 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b     | tpc-b_96.sql |  1000 |       8 |       2 |  955 |      11.603
  56 | Eff_csize 6G shrd_buff 2G maxp1000 chkpt_tout 300 ckpt_comp_tgt=0.9  archiveon tpc-b (2)               | tpc-b_96.sql |  1000 |       8 |       2 |  953 |      11.513
(20 rows)
~~~~


Sans surprise, les tests effectués avec le FILLFACTOR (32 et 33) arrivent en tête que ce soit pour la latence ou pour les tps. 
On trouve également le set 14 qui comporte une valeur `checkpoint_timeout` plus élevée (600s) et fait donc de ce set un bon candidat en terme de performance.

Lorsque vous aurez également un grand nombre de test, il vous faudra peut être également trouver le bon compromis et "fenêtrer" votre champ de vision 
et ainsi limiter votre horizon à ce qui vous semble le plus pertinent. 
Lorsque vous avez une idée précise de la taille de base de données et/ou du nombre de client habituel (qui peuvent varier au cours de la journée), cette recherche peut donc être facilitée avec ce script.



### 3. Cumul des paramètres et mesure différentielle

La façon de procéder par la suite est un peu délicate et nécessite de la réflexion.
On peut tenter d'analyser séparemment plusieurs jeux de paramètres pour voir s'il est possible de voir celui ou ceux qui seront les plus pertinent. 
La mesure différentielle nous permet de faire la différence entre chaque paramètre qui est changé.

Par la suite, je décide donc de cumuler certains paramètres dans des jeux séparés pour voir si ceux-ci peuvent se cumuler.
Après tout, nous cherchons à être pragmatique tout en restant méthodique.

Il nous font donc penser en terme de paramètres qui pourraient s'accomoder les uns des autres et améliorer ensemble les performances.
Tout d'abord, nous allons prendre comme témoin la configuration initiale (avec les erreurs de l'outil d'autoconfiguration corrigées).

Les paramètres les plus prometteurs étaient par ordre décroissant&nbsp;: 

1. le **FILLFACTOR**&nbsp;: nous allons prendre 95 pour minimiser l'impact disque;
2. le **checkpoint_timeout**&nbsp;: nous allons passer à 600s pour ne pas (trop) compromettre le temps de restauration en cas de crash;
3. le **checkpoint_target**&nbsp;: 0.95.

Pour déterminer plus facilement l'influence de chaque paramètre, je décide de n'en inclure que 2&nbsp;: `FILLFACTOR` et `checkpoint_timeout`.


# CONCLUSION & DISCUSSION

seconde partie

**Traitement statistique et méthode observationnelle**

Nous avons vu qu'il était possible de dégager une tendance rapidement grâce à l'observation.
De plus, il nous a été possible de déterminer de façon absolue quels étaient les sets les plus performants avec l'outillage de tri statistique.
Ceci nous permet de traiter un grand nombre de test.

<!--
Il me paraissait nécessaire d'améliorer le triage des tests.
C'est pour cela que j'ai ajouté les scripts permettant d'imposer des bornes supérieures et inférieures lors du filtrage des tests.
-->

Il serait intéressant de trier/comparer les sets de la même façon qu'ils sont filtrés pour le rapport.
Je pourrai inclure ceci sous forme de tableau avec comptage de points dans le rapport directement.
Je tenterai de faire ce développement utlérieurement.
<!-- Tendre vers d'avantage d'automatisation -->


Oops, I benched it again...