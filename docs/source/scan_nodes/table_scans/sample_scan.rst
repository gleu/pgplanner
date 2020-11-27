Sample Scan
===========

Parfois, il est intéressant de lire uniquement un échantillon d'une table. Le
standard SQL dispose pour cela de la clause ``TABLESAMPLE``. PostgreSQL
accepte cette clause et vous permet de préciser la méthode d'échantillonnage.

Voici un exemple d'une telle requête::

   SELECT * FROM t1 TABLESAMPLE SYSTEM(10)
   WHERE c1>10;

Et voici les informations renvoyées par l'instruction ``EXPLAIN``::

   Sample Scan on t1 [...]
     Sampling: system ('10'::real)
     Filter: (c1 > 10)
     Rows Removed by Filter: 999999
     Buffers: shared hit=4005 read=420

La première ligne correspond au nom du nœud, suivi du nom de la relation.
L'exécuteur va lire cette table, blocs par blocs, lire chaque enregistrement,
et garder uniquement les enregistrements visibles par la transaction en cours
et qui respectent le filtre (si un filtre est précisé). Il fera néanmoins un
filtre aléatoire respectant la méthode d'échantillonnage indiquée ainsi que la
fréquence d'échantillonnage.

La deuxième ligne indique la méthode d'échantillonnage.

La troisième ligne apparaît seulement quand l'exécuteur a besoin de filtrer
des données pour respecter un prédicat (une clause ``WHERE`` par exemple).
Elle précise le filtre réalisé (ici, il doit chercher toutes les lignes dont
la valeur de la colonne ``id`` est supérieure à 10).

La quatrième ligne apparaît seulement quand l'exécuteur a besoin de filtrer
des données pour respecter un prédicat et si l'option ``ANALYZE`` a été
utilisée avec l'instruction ``EXPLAIN``. Elle indique combien de lignes ont
été filtrées en appliquant le prédicat.

La cinquième ligne apparaît seulement si les options ``ANALYZE`` et
``BUFFERS`` sont utilisées. Elle indique comment les caches partagé, local et
les fichiers temporaire sont gérés : combien de blocs sont lus dans le cache
et en dehors, modifiés, écrits sur disque, etc.

