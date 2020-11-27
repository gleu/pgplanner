Seq Scan
========

La façon la plus traditionnelle et toujours fonctionnelle de lire les données
d'une table est de parcourir la table, bloc par bloc, séquentiellement, pour
décoder chaque ligne, les filtrer si nécessaire et les envoyer à l'utilisateur
(ou au nœud supérieur). D'où le terme de ``Sequential Scan`` (ou ``Seq
Scan``).

Ce type de parcours lit toujours tous les blocs de la table, sauf si la
requête a une clause ``LIMIT`` qui permettrait à l'exécuteur d'arrêter le
parcours avant. Il pourrait ne pas renvoyer toutes les lignes si un filtre
(tel qu'une clause ``WHERE``) est appliqué. Si une grande majorité de lignes
est filtrée, un parcours d'index pourrait être une meilleure option. Si une
grande majorité de lignes est renvoyée à l'utilisateur (ou au nœud supérieur),
un parcours séquentiel est probablement préférable.

Le plus gros avantage des parcours séquentiels est qu'ils fonctionnent
toujours. Qu'il y ait un index ou non, un parcours séquentiel est toujours
possible. C'est aussi le parcours le plus efficace si la relation est très
petite et si le nombre de lignes en résultat corresponds à une grande partie
du nombre total de lignes.

Comme les blocs sont lus dans l'ordre séquentiel, les lignes ne sont pas
spécialement triées. Elles sont lues dans l'ordre séquentiel physique, ce qui
ne signifie pas forcément qu'elles seront renvoyées dans leur ordre
d'insertion.

Voici les informations données par l'instruction ``EXPLAIN`` ::

   Seq Scan on t1 [...]
     Filter: (c1 = 10)
     Rows Removed by Filter: 999999
     Buffers: shared hit=4005 read=420

La première ligne indique le nom du nœud suivi du nom de la relation.
L'exécuteur va lire cette table, bloc par bloc, lire chaque ligne et ne
conserver que celles visibles pour la transaction en cours et respectant le
filtre (s'il y en a un).

La deuxième ligne apparaît seulement quand l'exécuteur a besoin de filtrer des
données pour respecter un prédicat (une clause ``WHERE`` par exemple).  Elle
précise le filtre réalisé (ici, il doit chercher toutes les lignes dont la
valeur de la colonne ``c1`` est égale à 10).

La troisième ligne apparaît seulement quand l'exécuteur a besoin de filtrer
des données pour respecter un prédicat et si l'option ``ANALYZE`` a été
utilisée avec l'instruction ``EXPLAIN``. Elle indique combien de lignes ont
été filtrées en appliquant le prédicat. Si ce nombre est bien plus important
que le nombre de lignes renvoyées par le nœud, un index pourrait aider à
diminuer la durée d'exécution de la requête.

La quatrième ligne apparaît seulement si les options ``ANALYZE`` et
``BUFFERS`` sont utilisées. Elle indique comment les caches partagé, local et
les fichiers temporaire sont gérés : combien de blocs sont lus dans le cache
et en dehors, modifiés, écrits sur disque, etc.

Quand PostgreSQL n'a pas de statistiques sur le nombre de lignes ou de blocs
(par exemple une table vide immédiatement après sa création), le planificateur
suppose qu'elle fait dix blocs, et estime le nombre de lignes dans ces dix
blocs. Donc l'estimation du coût final sera différent entre deux tables qui
diffèrent en nombre de lignes par bloc.

Voici comment créer la table avec laquelle nous allons expérimenter::

   CREATE TABLE t1 (c1 integer, c2 text);
   INSERT INTO t1 SELECT i, 'Line '||i FROM generate_series(1, 100000) i;
   ANALYZE t1;

Lisons la table complète. Un parcours séquentiel sera plus rapide car il lit
les pages séquentiellement. De plus, c'est la seule option actuellement vu que
la table n'a pas d'index::

   EXPLAIN SELECT * FROM t1;
   
                           QUERY PLAN                        
   -----------------------------------------------------------
    Seq Scan on t1  (cost=0.00..1541.00 rows=100000 width=14)
   (1 row)

Le coût dépend de différentes choses :

* le nombre de blocs ;
* le nombre de lignes ;
* et la configuration actuelle des paramètres ``seq_page_cost`` et
  ``cpu_tuple_cost``.

Nous pouvons obtenir le coût avec cette requête::

   SELECT
     round((
         current_setting('seq_page_cost')::numeric  * relpages
       + current_setting('cpu_tuple_cost')::numeric * reltuples
     )::numeric, 2)
   AS final_cost
   FROM pg_class
   WHERE relname='t1';

Exécuter cette requête renvoie comme résultat ``1541.00``, ce qui est en effet
le coût final.

S'il existe un filtre, cela coûtera plus parce qu'il devient nécessaire de
tester toutes les lignes de la table avec ce prédicat::

   EXPLAIN SELECT * FROM t1 WHERE c1=1000;
   
                        QUERY PLAN
   ------------------------------------------------------
    Seq Scan on t1  (cost=0.00..1791.00 rows=1 width=14)
      Filter: (c1 = 1000)
   (2 rows)

La ligne ``Filter`` montre le filtre réalisé. L'information ``rows=`` indique
le nombre de lignes que l'optimiseur pense renvoyer une fois le filtre
appliqué.

Le coût est plus important. Nous sommes passés de 1541 à 1791. Le coût
supplémentaire dépend du nombre de lignes dans la table et de la configuration
actuelle du paramètre ``cpu_operator_cost``. Le calcul du coût final de la
requête devient ::

   SELECT
     round((
         current_setting('seq_page_cost')::numeric     * relpages
       + current_setting('cpu_tuple_cost')::numeric    * reltuples
       + current_setting('cpu_operator_cost')::numeric * reltuples
     )::numeric, 2)
   AS final_cost
   FROM pg_class
   WHERE relname='t1';

ce qui nous donne un coût final de 1791.

Si l'opérateur est en fait une fonction utilisateur, le coût dépend de la
valeur de la clause ``COST`` de la fonction. L'exemple ci-dessus montre un coût
de 1791 avec l'opérateur d'égalité. Si nous écrivons une fonction PL/pgsql qui
utilise cet opérateur, mais qui a un coût de 10000, cela donnerait ceci ::

   CREATE FUNCTION equal(integer,integer)
   RETURNS boolean
   LANGUAGE plpgsql
   COST 10000
   AS $$
   BEGIN
     RETURN $1 = $2;
   END
   $$;
   
   EXPLAIN SELECT * FROM t1 WHERE equal(c1, 1000);
   
                            QUERY PLAN
   -------------------------------------------------------------
    Seq Scan on t1  (cost=0.00..2501541.00 rows=33333 width=14)
      Filter: equal(c1, 1000)
   (2 rows)

Le coût a réellement explosé à cause de la clause ``COST`` configurée sur la
fonction.

Concernant le nombre de lignes, le planificateur s'attend à en obtenir 33333
après avoir appliqué le filtre ``equal(c1, 1000)``. Pour savoir combien de
lignes sont réellement supprimées par le filtre, nous avons besoin d'exécuter
la requête, ce qui signifie utiliser l'option ``ANALYZE``::

  EXPLAIN (ANALYZE, BUFFERS)
    SELECT * FROM t1 WHERE equal(c1, 1000);
  
                             QUERY PLAN
  ------------------------------------------------------------
   Seq Scan on t1  (cost=0.00..2501541.00 rows=33333 width=4)
                   (actual time=3.157..44.679 rows=1 loops=1)
     Filter: equal(c1, 1000)
     Rows Removed by Filter: 99999
     Buffers: shared hit=541
   Planning Time: 0.081 ms
   Execution Time: 44.711 ms
  (6 rows)

Le paramètre ``enable_seqscan`` nous permet d'activer ou de désactiver les
parcours séquentiels. En fait, il n'est pas possible de désactiver totalement
les parcours séquentiels (tout simplement parce qu'il n'y a aucun autre moyen
de parcourir une table s'il n'y a pas d'index sur cette table). La conséquence
de la pseudo désactivation des parcours séquentiels est d'ajouter 10^10 au
coût, ce qui fait qu'on obtiendra tout de même un parcours séquentiel quand il
n'y a aucun moyen de procéder autrement, mais avec un coût délirant::

   SET enable_seqscan TO off;
   EXPLAIN SELECT * FROM t1 WHERE equal(c1, 1000);
                                   QUERY PLAN
   --------------------------------------------------------------------------
    Seq Scan on t1  (cost=10000000000.00..10002501541.00 rows=33333 width=4)
      Filter: equal(c1, 1000)
   (2 rows)
   RESET enable_seqscan;

