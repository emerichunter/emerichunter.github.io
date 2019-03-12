+++
title = "Il ne peut y en avoir qu'un"
banner = "banners/placeholder.png"
tags = ["administration"]
shortstory = "Bonne pratique sur la mise en place du fichier `postgresql.conf`. Les pièges à éviter, les fausses bonnes idées et les amis qui vous veulent du mal."
description = "Les mauvaises et les bonnes pratiques sur le paramétrage avec le fichier `postgresql.conf`"
draft = true
author = "Emeric Tabakhoff"
date = "2017-10-20T17:12:52+02:00"
categories = ["technique"]

+++


Aujourd'hui, je vais vous parler de bonnes pratiques.
Nous allons voir ensemble les bonnes raisons de ne pas faire de multiples fichiers de configuration de votre instance.

Certes, PostgreSQL est très flexible de ce point de vue et c'est une bonne chose, mais l'est-ce réellement ?
Ce n'est pas parce que vous pouvez faire de l'include de fichier que c'est nécessairement une bonne pratique.

## Mauvaises pratiques

* La lecture est séquentielle : le fichier est lu de manière séquentielle au démarrage de l'instance ou au rechargement de la configuration, c'est à dire dans l'ordre où sont écrits les paramètres. 
Si l'on fait une modification, celle-ci est réécrite par les fichiers dans l'include si ceux-ci sont à la fin.
On peut donc buter dessus un moment en tant que débutant lorsqu'on en a peu l'habitude.

* De même, si on met les fichiers au début (pour qu'ils soient plus visible !) et qu'une modification a bien lieu dans le bon fichier. 
Si le meme paramètre prend une autre valeur dans le corps du fichier ou dans un autre fichier, la valeur est réécrite.

   **LA DERNIERE VALEUR DU FICHIER EST DONC CELLE QUI EST PRISE EN COMPTE**

* On ne voit pas forcément les dits fichiers : même en faisant de gros efforts, il est toujours possible de passer à côté car ce n'est pas une pratique universelle.
Je me suis déjà heurté à ce moment d'intervention ou je change un paramètre en m'étant assuré qu'il s'agissait bien du bon fichier dans la configuration de l'instance.
Mais à chaque redémarrage, le paramètre garde la même valeur et la mienne n'est jamais prise en compte.
Je change un autre paramètre pour voir, cette fois il est modifié.
Un troisième, et il n'est pas modifié.
Je parcours tout le fichier postgresql.conf pour constater à la dernière ligne qu'il comporte plusieurs include : la valeur que je n'arrive pas à modifier est dans l'un d'eux.
**ajouter une commande pour lister les include**

* ça nuit à la lisibilité : devoir entrer dans les fichiers et faire des allers-retours pour trouver la valeur recherchée peut s'avérer chronophage.
Une partie des paramètre des wal dans un fichiers, une autre ailleurs et pour finir, des modifications également dans le corps du fichier.
Ce n'est pas une définition du rangement.
Les choses doivent être ordonnées sans quoi, un temps précieux est perdu.
Toujours penser au confrères qui arriveront après.

* Maintenabilité : c'est ici la même chose. 
C'est difficile à suivre et à lire donc à maintenir.
Un fichier est plus simple qu'un fichier principal plus 4 annexes.



## Bonnes pratiques

* Un seul fichier
* Structuré autour des axes principaux (pour la lisibilité et la maintenabilité) : wal, logging, memoire...
* Plusieurs fichiers de configuration quand même : Parce que dans certains cas, oui il est utile.
Lorsque l'on veut faire une session différentes des autres.
Particulièrement dans le cas d'une session de batch : besoin en `work_mem` différents (d'autres exemples de paramètres ?).
Pour les requêtes qui créént trop de fichiers temporaires.
Cela permet aussi de ne pas faire accéder aux développeurs ce genre de hacks.
Ils peuvent connaitre les paramètres de session mais utiliser tout de même un framework bien établi qui n'est pas figé mais qui permet de tracer les modifications.
Il est toujours possible de faire des include, mais surtout cela permet de garder un fichier principal et les configurations exotiques séparées.
Il faut donc que cela une réelle utilité.




