Index Only Scan
===============

Nous avons dit plus tôt que, lors d'un parcours d'index, la table est aussi
lue, même si ce n'est qu'une petite partie de cette table. Cette lecture de
table est utile pour obtenir les informations de visibilité, et les valeurs
des autres colonnes.

Pour les requêtes simples, l'exécuteur n'a pas besoin de connaître les valeurs
des autres colonnes. De plus, depuis la version 9.2, il existe un fichier
appelé la ``Visibility Map``, contenant des informations sur la visibilité des
enregistrements. Ceci signifie que, parfois, il n'est pas nécessaire de
parcourir la table lors d'un parcours d'index. Quand l'optimiseur pense que
c'est une bonne idée, il utilise un autre nœud appelé ``Index Only
Scan``. Par exemple, sur cette requête ::

  SELECT c1 FROM t1 WHERE c1<10;

L'index sera capable de filtrer les lignes très efficacement et d'avoir
suffisamment d'informations pour sortir les valeurs nécessaires. D'où ce
nouveau plan ::

   EXPLAIN SELECT c1 FROM t1 WHERE c1<10;

                                  QUERY PLAN                                
   -------------------------------------------------------------------------
    Index Only Scan using t1_c1_idx on t1  (cost=0.29..4.45 rows=9 width=4)
      Index Cond: (c1 < 10)
   (2 rows)

Voici les informations obtenues à partir de l'instruction ``EXPLAIN`` ::

   Index Only Scan using t1_id_idx on t1 ...
     Index Cond: (id < 10)
     Heap Fetches: 0
     Buffers: shared hit=4

La première ligne nous donne le nom du nœud, le nom de l'index et le nom de la
relation. L'exécuteur lira cet index, en suivant la structure de l'arbre sur
disque. À chaque valeur trouvée, il va vérifier si le bloc de cette ligne est
marqué ``tous visibles`` dans la ``Visibility Map``. Si le bloc est marqué
``tous visibles``, l'exécuteur sait que la ligne est visible et renverra la
valeur de la colonne.  Si ce n'est pas le cas, il lira la ligne de la table
pointé par l'index pour obtenir les informations de visibilité.

La ligne 2 apparaît seulement quand l'index peut satisfaire un prédicat. Dans
cet exemple, l'exécuteur utilise l'index pour trouver rapidement les lignes
pour lesquelles c1 est inférieur à 1000.

La troisième ligne apparaît seulement quand la clause ``ANALYZE`` est utilisée
avec l'instruction ``EXPLAIN``. Elle indique le nombre de lignes lues dans la
table.

La quatrième ligne apparaît seulement si les clauses ``ANALYZE`` et
``BUFFERS`` sont utilisées. Elle indique comment sont utilisés le cache
partagé, le cache local et les fichiers temporaires : combien de blocs sont
lus dans le cache et hors du cache, modifiés, écrits sur disque, etc.

Voyons un exemple. Lisons ``t1`` et récupérons toutes les lignes pour
lesquelles la valeur de la colonne ``c1`` est supérieure à 90000 ::

   EXPLAIN (ANALYZE, BUFFERS) SELECT c1 FROM t1 WHERE c1>90000;

                                      QUERY PLAN
   -------------------------------------------------------------------------------
    Index Only Scan using t1_c1_idx on t1  (cost=0.29..297.31 rows=10344 width=4)
                                    (actual time=0.052..3.999 rows=10000 loops=1)
      Index Cond: (c1 > 90000)
      Heap Fetches: 0
      Buffers: shared hit=3 read=28
    Planning Time: 0.110 ms
    Execution Time: 5.327 ms
   (6 rows)

La ligne ``Heap Fetches`` nous indique que l'exécuteur n'a pas eu besoin de
lire de lignes dans la table. Maintenant, ajoutons quelques lignes dans la
table, et testons de nouveau ``EXPLAIN`` ::

   INSERT INTO t1 SELECT i, 'Line '||i FROM generate_series(100001, 120000) i;
   EXPLAIN (ANALYZE, BUFFERS) SELECT c1 FROM t1 WHERE c1>90000;

                                      QUERY PLAN
   -------------------------------------------------------------------------------
    Index Only Scan using t1_c1_idx on t1  (cost=0.29..372.45 rows=12409 width=4)
                                   (actual time=0.042..24.687 rows=30000 loops=1)
      Index Cond: (c1 > 90000)
      Heap Fetches: 20100
      Buffers: shared hit=194
    Planning Time: 0.212 ms
    Execution Time: 29.959 ms
   (6 rows)

L'insertion a ajouté quelques blocs dans la table, et il n'y a pas eu de
``VACUUM`` entre temps, ce qui signifie que la ``Visibility Map`` n'a pas été
mise à jour. Donc, maintenant, la ligne ``Heap Fetches`` indique que 20100
lignes ont été lues dans la table pour déterminer leur visibilité pour la
transaction en cours.

Maintenant, exécutons un ``VACUUM`` et essayons de nouveau ::

   VACUUM t1;
   EXPLAIN (ANALYZE, BUFFERS) SELECT c1 FROM t1 WHERE c1>90000;

                                      QUERY PLAN
   -------------------------------------------------------------------------------
    Index Only Scan using t1_c1_idx on t1  (cost=0.29..357.36 rows=12404 width=4)
                                   (actual time=0.058..15.885 rows=30000 loops=1)
      Index Cond: (c1 > 90000)
      Heap Fetches: 0
      Buffers: shared hit=85
    Planning:
      Buffers: shared hit=11
    Planning Time: 0.360 ms
    Execution Time: 21.810 ms
   (8 rows)

``Heap Fetches`` est de retour à zéro parce que la ``Visibility Map`` sait que
toutes les lignes des blocs lus étaient visibles.

Les index B-Tree sont les premiers à supporter ce type de nœud. Les index
GiST et SP-GiST sont compatibles depuis la version 9.5, mais seulement pour
certaines classes d'opérateur. Les index GIN ne sont pas compatibles avec ce
nœud. Les index BRIN ne seront jamais compatibles parce qu'ils ne contiennent
pas les valeurs, seulement des intervalles de valeurs par bloc. Les index
partiels supportent ce type de nœud depuis la version 9.6.

Ce type de parcours est particulièrement efficace sur les tables peu modifiées. Il
sera plus facilement sélectionné si vous prenez soin de l'ordre des colonnes de
l'index : tout d'abord celles qui sont utilisées dans les clauses WHERE des requêtes,
puis celles qui sont uniquement utilisées dans les clauses SELECT .

Certaines requêtes ne permettent pas de bénéficier de ce type de parcours parce
qu'elles demandent d'accéder à des colonnes qui ne font pas partie de l'index. Il
est possible de les inclure dans l'index sans qu'elles ne fassent partie de la clé, ce
qui est très intéressant dans le cas d'un index pour une contrainte d'unicité ou une
clé primaire. Il faut pour cela utiliser la clause INCLUDE qui apparaît en version 11.
On parle alors d'index couvrant.

Le paramètre enable_indexonlyscan permet de désactiver temporairement les par-
cours d'index couvrant. Il est essentiel de ne pas les désactiver globalement en pro-
duction.
