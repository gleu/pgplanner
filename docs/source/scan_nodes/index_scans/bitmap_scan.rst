Bitmap Scan
===========

Un des plus gros problèmes avec le nœud ``Index Scan`` est le déplacement de
la tête du disque lors du parcours de l'index et celui de la table. Chaque
enregistrement intéressant sur l'index est immédiatement vérifié sur la table.
Autrement dit, PostgreSQL passe son temps à lire un peu l'index, puis un peu
la table, de façon répétée. Ceci est très mauvais pour les performances.

Les développeurs de PostgreSQL ont essayé de diminuer l'impact sur les
performances en parcourant tout d'abord et entièrement l'index, et en stockant
les pointeurs d'enregistrements en mémoire. Une fois l'index lu, il est rapide
et facile de parcourir la table, séquentiellement, d'après la liste de
pointeurs en mémoire, dans l'ordre physique de la table pour éviter au maximum
les déplacements aléatoires de la tête de lecture.

Ce type de parcours est appelé un ``Bitmap Index Scan`` parce qu'il crée un
index bitmap en mémoire pendant le parcours de l'index. Ce bitmap est un champ
de bits. Une valeur 0 signifie que l'enregistrement ne correspond pas au
filtre, et une valeur 1 signifie qu'il correspond au filtre. S'il y a trop
d'enregistrements à stocker, le bitmap devient ``lossy`` (à perte), et chaque
bit représente un bloc, plutôt qu'un enregistrement. Dans un tel cas, pendant
l'étape du parcours de la table, une vérification supplémentaire (un
``recheck``) est fait au niveau du bloc. L'exécuteur sait qu'une ou plusieurs
lignes satisfont le filtre mais il a besoin de savoir exactement lesquelles.
L'avantage est que l'exécuteur peut fonctionner sur un énorme nombre de lignes
sans utiliser trop de mémoire.

Comme la construction de ce bitmap se fait en parcourant un index, les
informations n'arrivent pas dans l'ordre physique des lignes et blocs de la
table mais dans l'ordre physique de l'index, ce qui n'est pas idéal pour créer
un bitmap trié tel qu'on le voudrait. Afin de créer cette structure de manière
efficace, la construction se fait en deux temps. Lors du parcours initial de
l'index, la partie du bitmap correspondant au bloc de la ligne à mémoriser est
d'abord stockée dans une table de hachage.  Cela permet de retrouver cette
même partie très rapidement pour les prochaines lignes correspondant au même
bloc. Une fois que le parcours initial de l'index est effectué, la table de
hachage est parcourue séquentiellement par ordre croissant de sa clé,
c'est-à-dire par ordre croissant des blocs de la table correspondante. Il
suffit donc de chaîner les différentes parties du bitmap pour générer le
bitmap final prêt à être utilisé.

La table de hachage utilisée lors de la création d'un bitmap a quelques
particularités par rapport à d'autres tables de hachage utilisées par
PostgreSQL. Tout d'abord, elle aura toujours la même structure. Ainsi, la clé
de la table de hachage est toujours la même, de même que la fonction
permettant de comparer cette clé. De plus, il n'est pas rare de se retrouver à
stocker un très grand nombre d'entrées. Or, PostgreSQL dispose d'une
implémentation généraliste de table de hachage, appelée ``dynahash``,
permettant de gérer beaucoup de cas possibles. Mais celle-ci n'est pas très
performante dans ce cas particulier. C'est pourquoi un autre type de table de
hachage a été introduit avec la version 10 de PostgreSQL. Cette nouvelle
implémentation, appelée ``simplehash``, permet de définir à la compilation et
non à l'exécution le type de la clé ainsi que les différentes fonctions
nécessaires pour utiliser cette table de hachage, ce qui évite un surcoût
inutile. De plus, la gestion des collisions est pensée pour être efficace dans
le cas de très grosses tables de hachage, où les collisions sont fréquentes.
Tout cela contribue à grandement améliorer les performances de ce nœud en
version 10.

Il est à noter que, contrairement à d'autres moteurs de base de données,
PostgreSQL ne permet pas de stocker l'index bitmap sur disque. Ce sont
uniquement des représentations en mémoire des index existants.

Ce type de parcours est composé d'au moins deux nœuds. Le premier, un ``Bitmap
Index Scan`` fait le parcours de l'index et crée l'index bitmap en mémoire. Le
deuxième, appelé ``Bitmap Heap Scan``, lit les blocs pointés par l'index
bitmap dans la table pour vérifier le respect du prédicat des lignes de ce
bloc. Ainsi, il récupère les lignes à renvoyer. En voici un exemple simple::

   Bitmap Heap Scan on t1 ...
     Recheck Cond: (c1 = ANY ('{10,40,60,100,600}'::integer[]))
     -> Bitmap Index Scan on t1_c1_idx ...
        Index Cond: (c1 = ANY ('{10,40,60,100,600}'::integer[]))

Le parcours de table étant séquentiel et partiel (uniquement pour les blocs
intéressants), il est bien plus rapide qu'un parcours d'index comme le ``Index
Scan`` pour un nombre moyen de lignes.

Donc son premier intérêt est purement la performance du parcours due au
découplage des parcours de l'index et de la table. Il présente néanmoins un
autre intérêt.

Soit une requête utilisant un filtre sur deux colonnes::

   SELECT * FROM t1 WHERE c1<100000 AND c2>=50

Une autre force de ce type de parcours est qu'il est très facile de combiner
plusieurs de ces bitmaps pour récupérer les lignes voulues. En effet, s'il
existe un index sur ``c1`` et un index sur ``c2``, PostgreSQL ne pourra pas
faire un parcours d'index sur ces deux index à la fois. Cependant, combiner
deux champs de bits en mémoire est très efficace. Il s'agit d'opérations
``AND`` ou ``OR`` booléennes très simples et très rapides.  Dans ce cas,
l'exécuteur calcule un index bitmap pour l'index de ``c1`` et un index bitmap
pour l'index de ``c2``. Ceci fait, il ne lui reste plus qu'à faire un ``AND``
logique (nœud ``BitmapAnd``) sur les deux champs de bits s'il faut retourner
les lignes correspondant aux deux prédicats, ou un ``OR`` logique (nœud
``BitmapOr`` ) pour les lignes correspondant à au moins un prédicat, et il se
retrouve avec un unique champ de bits qu'il sait traiter avec un nœud ``Bitmap
Heap Scan``::

   Bitmap Heap Scan on t1 ...
     Recheck Cond: ((c1 < 100000) AND (c2 > 50))
     Rows Removed by Index Recheck: 4433369
     Heap Blocks: exact=19820 lossy=19839
     Buffers: shared read=39935
     -> BitmapAnd ...
        -> Bitmap Index Scan on t1_c1_idx ...
           Index Cond: (c1 < 100000)
           Buffers: shared read=39935
        -> Bitmap Index Scan on t1_c2_idx
           Index Cond: (c2 > 50) ...
           Buffers: shared read=39935

``Bitmap Heap Scan`` est le nom du nœud parent. Il s'occupe de renvoyer les
données vérifiées auprès de la table.

La ligne ``Recheck Cond`` indique que le filtre complet est vérifié au niveau
de la table. Cette opération est nécessaire lorsque le nombre de lignes est
trop important et qu'on est passé d'un index bitmap d'enregistrements à un
index bitmap de blocs. Ce changement de type de bitmap est très préjudiciable
pour les performances. Une augmentation de la valeur du paramètre ``work_mem``
est le seul moyen de l'éviter.

La ligne ``Rows Removed by Index Recheck`` indique le nombre de lignes
supprimées par la revérification. Elle n'apparaît que si l'instruction
``EXPLAIN`` utilise l'option ``ANALYZE``.

La ligne ``Heap Blocks`` indique le nombre de blocs exacts et le nombre de
blocs à perte. S'il y a des blocs à perte, cela sous-entend que la valeur du
paramètre ``work_mem`` n'est pas suffisamment haute pour avoir un bitmap de
lignes. Une augmentation de la valeur de ``work_mem`` permettrait d'améliorer
les performances des requêtes.

Les lignes ``Buffers`` apparaissent seulement si les clauses ``ANALYZE`` et
``BUFFERS`` sont utilisées. Elles indiquent combien de blocs sont utilisés
dans le cache partagé, dans le cache local et pour les fichiers temporaires :
combien de blocs sont lus dans le cache et hors du cache, modifiés en mémoire,
écrits sur disque, etc.

La ligne ``BitmapAnd`` indique qu'une opération ``AND`` booléenne est exécutée
entre deux index bitmaps. Les deux parcours d'index bitmaps sont les enfants
de ce nœud.

Il est à noter qu'il existe aussi le nœud ``BitmapOr`` quand le lien entre deux
filtres est un ``OR`` booléen.

Ces deux lignes sont les enfants directs de la ligne ``BitmapAnd``. Ce sont
des parcours d'index qui sont représentés en mémoire par un champ de bits.
Chacun des bits correspond à une ligne (ou à un bloc si le nombre de lignes
est trop important).

La ligne ``Index Cond`` indique le filtre utilisé. Dans le cas de cette ligne,
PostgreSQL utilise l'index ``t1_c1_idx`` pour trouver rapidement les lignes
satisfaisant le filtre ``c1 < 1000``.

Du fait d'avoir à d'abord créer l'index bitmap en mémoire, la récupération des
premières lignes est lentes (on dit que le coût de démarrage est élevé), ce
parcours est moins intéressant dans le cas de l'utilisation de curseurs ou de
la clause ``LIMIT``. Son efficacité est augmentée si les disques utilisés
peuvent gérer un grand nombre d'opérations d'entrées/sorties disque en
parallèle. Dans ce cas, il faut configurer le paramètre
``effective_io_concurrency`` avec une valeur supérieure à zéro. Pour des
disques SSD, la valeur peut atteindre plusieurs centaines.

Si le nombre de blocs à perte est positif, la valeur du paramètre ``work_mem``
n'est pas suffisante. Prenons l'exemple suivant::

   CREATE TABLE aa AS
     SELECT * FROM generate_series(1, 10000000) AS a ORDER BY random();
   CREATE INDEX aai ON aa(a);
   -- pour éviter les parcours Index Scan et Seq Scan
   SET enable_indexscan TO false;
   SET enable_seqscan TO false;
   SET max_parallel_workers_per_gather to 0;

Ici, un ``Bitmap Scan`` a tout son sens. Diminuons la valeur de ``work_mem`` à
sa valeur minimale::

   SET work_mem TO '64kB';
   EXPLAIN (analyze,buffers) SELECT * FROM aa WHERE a BETWEEN 100000 AND 200000;

                                                           QUERY PLAN
   --------------------------------------------------------------------------------------------------------------------------
    Bitmap Heap Scan on aa  (cost=2081.70..194610.72 rows=98075 width=4) (actual time=33.456..987.185 rows=100001 loops=1)
      Recheck Cond: ((a >= 100000) AND (a <= 200000))
      Rows Removed by Index Recheck: 8784375
      Heap Blocks: exact=349 lossy=39310
      Buffers: shared read=39935
      ->  Bitmap Index Scan on aai  (cost=0.00..2057.18 rows=98075 width=0) (actual time=33.269..33.269 rows=100001 loops=1)
            Index Cond: ((a >= 100000) AND (a <= 200000))
            Buffers: shared read=276
    Planning Time: 0.161 ms
    Execution Time: 991.343 ms
   (10 rows)

On récupère ici 349 blocs exacts et 39310 blocs à perte. Pour connaître la
bonne valeur du paramètre ``work_mem``, il suffit d'utiliser cette formule::

   (nombre blocs à perte + nombre de blocs exacts) * (48 + 8 + 8)

ce qui nous donne 2,5 Mo. Vérifions cela::

   SET work_mem TO '2.5MB';
   EXPLAIN (analyze,buffers) SELECT * FROM aa WHERE a BETWEEN 100000 AND 200000;

                                                           QUERY PLAN
   --------------------------------------------------------------------------------------------------------------------------
    Bitmap Heap Scan on aa  (cost=2081.70..127583.55 rows=98075 width=4) (actual time=46.241..236.572 rows=100001 loops=1)
      Recheck Cond: ((a >= 100000) AND (a <= 200000))
      Heap Blocks: exact=39659
      Buffers: shared read=39935
      ->  Bitmap Index Scan on aai  (cost=0.00..2057.18 rows=98075 width=0) (actual time=38.026..38.026 rows=100001 loops=1)
            Index Cond: ((a >= 100000) AND (a <= 200000))
            Buffers: shared read=276
    Planning Time: 0.160 ms
    Execution Time: 240.751 ms
   (9 rows)

Nous navons là que des blocs exacts, et la durée d'exécution a été divisée par
4.

Attention, la formule de calcul est différente pour les versions antérieures à
la version 10. Voici la formule::

   (nombre blocs à perte + nombre de blocs exacts) * (16 + 48 + 8 + 8)

Ceci est dû au changement de la méthode de hachage.

Le paramètre ``enable_bitmapscan`` permet de désactiver temporairement les
parcours d'index bitmap. Il est essentiel de ne pas les désactiver globalement
en production.
