+++
shortstory = "Présentation des procédures stockées, une nouvelle fonctionnalité dans PostgreSQL version 11"
description = "Avec PostgreSQL 11 qui arrive, faisons un tour d'horizon de ce qui nous attend avec les procédures stockées."
title = "Les Procédures stockées dans PostgreSQL 11"
author = "Emeric TABAKHOFF"
banner = "head/placeholder.png"
date = "2018-05-14T10:36:11+02:00"
tags = ["administration","beta","code","sql"]
categories = ["technique","veille"]
draft = true

+++

<!-- # Transactions autonomes et PROCEDURES STOCKEES dans PostgreSQL 11 -->

Notes de départ :
http://blog.dalibo.com/2016/08/19/Support_des_transactions_autonomes_dans_PostgreSQL.html

La sortie de PostgreSQL version 11 est pour la rentrée prochaine. 
Certaines des nouvelles fonctionnalités sont dors et déjà annoncées.
L'une d'entre elle promet de changer radicalement la façon de procéder avec le code SQL dans l'avenir. 
Que vous ayez migré d'un SGBD inclant auparavant les procédures stockées et utilisé ou non les transaction autonomes ou que vous utilisiez déjà Postgres, 
je pense que cette nouvelle fonctionnalité mérite de s'y intéresser.
Ce qui change pour cette version peut, non seulement avoir des implications sur la façon dont vous utilisiez le PL/pgSQL jusqu'ici,
mais également vous amener à modifier la manière dont les procédures stockées et les transactions autonomes ont été implémentées lors de votre migration. 


## Introduction 

### Principe de la transaction autonome


<!-- reformuler : attention aux redites -->
* Une transaction principale
* Une ou plusieurs transactions indépendantes déclenchées lors de l'exécution de la transaction principale
* Controle du commit/rollback au sein de la procédure des différentes sous transactions de façon indépendante
* Autorise donc les transactions dites autonomes, transaction au commit/rollback indépendant du corps de la transaction principale.
* La transaction principale n'est pas affectée en cas d'echec d'une sous transaction.
* La transaction principale ne voit pas non plus le résultat des opérations réalisées par les transactions autonomes avant le commit final


### Transaction autonome et ACIDidité

Attendez une seconde... Commit indépendant ? Mais que devient l'acidité garantie en temps normal par la transaction principale ? 
D'un point de vue absolu, on souhaite garder la cohérence des données dans la base sur laquelle s'effectue nos transactions. 
Avoir seulement une ou plusieurs sous transactions autonomes reussies à l'intérieur d'une transaction principale elle-même en échec semble _a priori_ contre intuifif et aller à l'encontre de toute logique. 
Penchons-nous sur des cas pratiques pour mieux appréhender l'utiilité d'une pareille fonctionnalité.

## Cas pratique

- Table d'audit/table de tracage des actions "tentées" par la procédure stockée

~~~sql
CREATE TABLE tablelog (id serial, etat varchar, dateoperation timestamptz DEFAULT now());
CREATE TABLE tabletest (id serial, calculcomplex numeric(0,2) );


CREATE PROCEDURE p_transaction_log()
 AS $$
 BEGIN
/* Log de debut d'operation */
      BEGIN
          INSERT INTO tablelog (etat, ) VALUES ('debut');
      END
/* operation */
      BEGIN
          WHILE LOOP
          END LOOP;
          
      END
/* Log de fin d'operation */
      BEGIN
          INSERT INTO tablelog (etat) VALUES ('fin');  
      END
     
 END
 $$
 LANGUAGE PLPGSQL; 
~~~

Ici, le début et la fin de l'opération effectuée par la procédure stockée sont logués dans une table indépendamment de la réuissite de l'opération principale. 



- "IF SUCCESS COMMIT; ELSE ROLLBACK; END" : controle plus fin qui permet d'aller au-delà de la seule prise en compte des erreurs.

~~~sql
CREATE TABLE table1 (entier int);

CREATE PROCEDURE p_transaction_test()
 AS $$
 BEGIN
     FOR i IN 0..9 LOOP
         INSERT INTO table1 (entier) VALUES (i);
         IF i % 2 = 0 THEN
             RAISE NOTICE 'i=%, txid=% will be committed', i, txid_current();
             COMMIT;
         ELSE
             RAISE NOTICE 'i=%, txid=% will be rolledback', i, txid_current();
             ROLLBACK;
         END IF;
     END LOOP;
 END
 $$
 LANGUAGE PLPGSQL; 
~~~

Cette fois, la procédure permet de committer ou rollbacker en fonction du résultat de l'expression conditionnelle. 

<!-- humpf bof -->
Les transactions autonomes permettent donc à la personne en charge du code SQL d'accéder à une granularité plus fine lors du comportement des différentes étapes d'une procédure stockée.
Ce qui simplifie d'autant la gestion des taches complexes par la base de données.
Bien sûr, une plus grande liberté implique une vigilance accrue vis à vis de ce type d'objet.

## Ancien fonctionnement dans PG - Anecdotique faire un résumé
- contournement \<9.5 : `dblink`
- contournement 9.5-10 : `pg_background`
- transformation par les outils de migration

~~ performances ~~

## Nouveau fonctionnement dans PG :

- creation CREATE/ALTER PROCEDURE
- appel :
  * CALL 
  * dans ou hors d'un SELECT
  * un seul jeu de résultat (pour le moment)
  * controle du COMMIT/ROLLBACK à l'intérieur de la transaction (v. plus haut)
~~- performances (?-> autre article)~~


## Différence fonction/procédure : 

### Functions :
* can be called inside a query (select func() from foo)
* generally return a result
* must return a single set
* are scoped to a transaction

### Procedures :
* can not be called inside a query
* typically don’t return results except for maybe error code
* can return multiple result sets (although FWICT the postgres implementation does not allow for this yet)
* **can flush the transaction (essentially a COMMIT; followed by a BEGIN;) within the procedure**

Being able to flush the transaction is by far the most important part; it allows for various kinds of things that are dangerous or impossible with functions (for example, a routine that never terminates).


J'espère que les outils de migrations s'adapterons dans le futur pour incorporer ce nouveau formalisme. Evitant ainsi les contournements évoqués plus haut. 
De cette façon, la maintenabilité sera grandement améliorée.