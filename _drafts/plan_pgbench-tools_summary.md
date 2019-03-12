# Article 0 - "Avant de démarrer"


# Article 1 - "Pour bien démarrer"
* Démarrage rapide
* Prerequis
* Installation
* mise en garde WAL
* Echelle & type de script
* Reproductibilité 
* VACUUM
* Use cases
* Ccl préliminaire
* "les outils" à changer
<!-- * Contexte & contraintes -->
<!-- * La machine -->
<!-- * les tests -->
<!-- * original & correctif 9.6 + use cases-->
<!-- * Problèmes rencontrés (fichiers fantômes - deleted) -->
<!-- * Conclusion : buffers & checkpoints -->

# Article 2 - "Pour comparer" 
* Contexte & contraintes : les paramètres modifiés
* La machine 
* les tests + graphs
  * regime heure de pointe
  	* paramètres et résultats
	* interprètations visuelles
	* outils de traitements /reports)
	* cumul des paramètres
  * régime "normal" (explication rapide)  ou dans la conclusion
* Problèmes rencontrés (fichiers fantômes - deleted)
* original & correctif 9.6

* Conclusion & Discussion :
   * scripts ajoutés : cleanup, limited_webreport, compromise, list_orderbytest
   * latence
   
   
***

checklist :


* Les problèmes pratiques&nbsp;: GNUPLOT (version), version PG-9.6, 
* le gd nb de test et la gestion des comparatifs, 
* suppression des erreur (humaines et issues de plantage)
* Les solutions/Contournements&nbsp;: lien vers la nouvelle version avec les nouveautés&nbsp;: tests 9.6, 
* listes en ordre, 
* cleanup, 
* graph pour certains sets uniquement (lisibilité), 
* redémarrage auto instance avant lancement du test, 
* graph de la latence
* ballooning ?
* fichiers fantome 

***

# Article 3 - "Pour aller plus loin"
* le régime "normal" : bench IRL
	* c'est quoi ?
	* comment ?
	* les tests et les paramètres 
	* TPS fixe, latence en fonction des tps. 
	* nouveaux graphs, nouveau rapport (rates_report)
	
* Problèmes rencontrés 
	* log-to-csv, csv2gnuplot

* reports/bufreport.sql
* buffers & checkpoints

* Conclusion
	* autre façon de faire du bench : la vie réelle

# Article 4 - "Vers l'infini et au-delà !"

* wrapper de wrapper
* reconfiguration auto
* reconfiguration on the fly
* Réplication : répartition de charge
* lru 
* Autovacuum
* fill factor
* retour duc checkpoint_timeout
* paramètres système

* L'auteur


Troisième partie&nbsp;: 
Pour comparer. 
Aujourd'hui, abordons un cas pratique. 
La méthode pour trouver les paramètres et les valeurs associées qui ont le plus d'impact sur votre configuration. 
Vous souhaitez améliorer votre configuration pour votre instance Postgres ? 
Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer ? 
Hélas, vous ne pouvez pas prévoir leur impact à l'avance. 
Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données ? 
Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."


description = "
Vous souhaitez améliorer votre configuration pour votre instance Postgres ? 
Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer ? 
Hélas, vous ne pouvez pas prévoir leur impact à l'avance. 
Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données ? 
Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."




Troisième partie&nbsp;: 
Pour comparer. 
Aujourd'hui, abordons un cas pratique. 
La méthode pour trouver les paramètres et les valeurs associées qui ont le plus d'impact sur votre configuration. 
Vous souhaitez améliorer votre configuration pour votre instance Postgres ? 
Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer ? 
Hélas, vous ne pouvez pas prévoir leur impact à l'avance. 
Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données ? 
Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."


description = "
Vous souhaitez améliorer votre configuration pour votre instance Postgres ? 
Vous connaissez les paramètres à changer (ou non) et les valeurs à essayer ? 
Hélas, vous ne pouvez pas prévoir leur impact à l'avance. 
Quelle est la meilleure configuration d'instance pour VOTRE configuration matérielle et votre utilisation de base de données ? 
Nous allons voir ensemble comment automatiser les tests de changement de configuration et estimer leur impact sur votre configuration matérielle avec pgbench-tools..."

