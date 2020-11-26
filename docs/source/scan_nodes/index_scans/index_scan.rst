Index Scan
==========

Le nœud ``Index Scan`` est la plus ancienne façon de parcourir un index.  Elle
est toujours d'actualité, mais il existe d'autres façons, plus optimisées, de
parcourir un index.

Quand l'exécuteur doit réaliser un parcours d'index, il va parcourir l'index à
la recherche des valeurs qui l'intéressent. Chaque fois qu'il trouve une bonne
valeur, il lit la ligne dans la table. Il doit faire cela parce que seul
l'enregistrement dans la table contient les informations de visibilités. Ces
informations ne font pas partie de l'index. De plus, il pourrait avoir besoin
de la valeur d'autres colonnes pour répondre à la requête.

Voici les informations renvoyées par l'instruction ``EXPLAIN`` ::

   Index Scan using t1_id_idx on t1 ...
     Index Cond: (c1 < 1000)
     Filter: (c2 = 'Line 30')
     Rows Removed by Filter: 66
     Buffers: shared hit=6

La première ligne indique le nom du nœud, le nom de l'index et le nom de la
table. L'exécuteur va lire cet index, en suivant sa structure d'arbre sur
disque. À chaque valeur trouvée, il va lire l'enregistrement associé au niveau
de la table.

La deuxième ligne apparaît seulement quand l'index peut satisfaire un
prédicat. Dans cet exemple, l'exécuteur utilise l'index pour trouver
rapidement les enregistrements pour lesquels la valeur de la colonne ``c1``
est inférieure à 1000.

La troisième ligne apparaît seulement si un autre filtre est utilisé sur la
relation de l'index. Ce filtre est réalisé quand une ligne est lue dans la
table, donc après le filtre de l'index.

La quatrième ligne apparaît seulement quand l'exécuteur a besoin de filtrer
des données autres que celes contenues dans l'index et si la clause
``ANALYZE`` est utilisée avec l'instruction ``EXPLAIN``. Elle indique le
nombre de lignes filtrées en appliquant le prédicat.

La cinquième ligne apparaît seulement si les clauses ``ANALYZE`` et
``BUFFERS`` sont utilisées. Elles indiquent combien de blocs sont utilisés
dans le cache partagé, dans le cache local et pour les fichiers temporaires :
combien de blocs sont lus dans le cache et hors du cache, modifiés en mémoire,
écrits sur disque, etc.

Donc, si nous avons un index sur notre table ``t1``, voici ce que donnerait un
parcours d'index sur cette table::

   CREATE INDEX ON t1(c1);
   EXPLAIN SELECT * FROM t1 WHERE c1=1000;

                                QUERY PLAN
   ---------------------------------------------------------------------
    Index Scan using t1_c1_idx on t1  (cost=0.29..8.31 rows=1 width=14)
      Index Cond: (c1 = 1000)
   (2 rows)

Un nœud ``Index Scan`` peut aussi être utilisé pour trier des donénes, par
exemple lors de l'utilisation d'une clause ``ORDER BY``. Il faut cependant que
l'index soit un B-Tree. Les données dans un index B-Tree sont déjà triées,
donc l'exécuteur a juste besoin de lire les données dans l'ordre ::

   EXPLAIN SELECT * FROM t1 ORDER BY c1;

                                    QUERY PLAN
   -----------------------------------------------------------------------------
    Index Scan using t1_c1_idx on t1  (cost=0.29..3148.29 rows=100000 width=14)
   (1 row)

Si l'exécuteur a besoin de la plus grosse valeur en premier, il peut faire un
parcours inversei ::

   EXPLAIN SELECT * FROM t1 ORDER BY c1 DESC;

                                         QUERY PLAN
   --------------------------------------------------------------------------------------
    Index Scan Backward using t1_c1_idx on t1  (cost=0.29..3148.29 rows=100000 width=14)
   (1 row)

Ces deux exemples ne fonctionneront qu'avec des index B-Tree. Les autres
méthodes d'accès aux index ne peuvent pas être utilisés pour ça.

Dû à la structure d'arbre stockée dans un index, réaliser un parcours d'index
est très lent parce que le système d'exploitation a besoin de déplacer les
têtes de lecture du disque pour trouver le prochain bloc à utiliser. Ce
comportement tend à désactiver la fonctionnalité de ``Read Ahead`` du système
d'exploitation. De ce fait, lire la même quantité de données dans une table et
dans un index aura des performances totalement différentes. Le parcours
d'index sera plus long, sauf si l'index est disponible en mémoire ou sur un
disque SSD. Dans ces deux derniers cas, il n'est pas nécessaire de déplacer
des têtes et, de ce fait, le parcours est plus rapide.

Le paramètre ``random_page_cost`` correspond au coût associé à la lecture d'un
bloc dans un index. Ce coût est plus important que ``seq_page_cost`` parce
qu'il est nécessaire de déplacer la tête du disque pour lire le prochain bloc.
Par détaut, la valeur de ``random_page_cost`` est quatre fois plus haute que
celle de ``seq_page_cost``. Diminuer ``random_page_cost`` motivera le
planificateur à choisir un parcours d'index car cela diminuera le coût total
d'utilisation de l'index. Voici un exemple avec deux valeurs différentes pour
ce paramètre ::

   SET random_page_cost TO 4;
   EXPLAIN SELECT * FROM t1 ORDER BY c1 DESC;

                                         QUERY PLAN
   --------------------------------------------------------------------------------------
    Index Scan Backward using t1_c1_idx on t1  (cost=0.29..3148.29 rows=100000 width=14)
   (1 row)

   SET random_page_cost TO 1;
   EXPLAIN SELECT * FROM t1 ORDER BY c1 DESC;

                                         QUERY PLAN
   --------------------------------------------------------------------------------------
    Index Scan Backward using t1_c1_idx on t1  (cost=0.29..2317.29 rows=100000 width=14)
   (1 row)

En diminuant ``random_page_cost`` de 4 à 1, le coût total est bien plus bas.

Le paramètre ``enable_indexscan`` nous permet d'activer ou de désactiver ce
nœud. Il est fortement déconseillé de le désactiver globalement.
