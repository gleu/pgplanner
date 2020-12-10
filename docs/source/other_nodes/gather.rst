Gather
======

Ce nœud est utilisé dans le cadre du parallélisme. Il a pour but de récupérer
les ensembles de données en sortie des opérations parallélisées et fournit un
ensemble de données en sortie qui est une simple agrégation des résultats.

Voici les informations données par l'instruction ``EXPLAIN`` ::

   Gather [...]
     Workers Planned: 4
     Workers Launched: 4
     Buffers: shared hit=4005 read=420
     -> Parallel Seq Scan

La première ligne indique le nom du nœud suivi du nom de la relation.
L'exécuteur va lire cette table, bloc par bloc, lire chaque ligne et ne
conserver que celles visibles pour la transaction en cours et respectant le
filtre (s'il y en a un).

La deuxième ligne indique le nombre de workers planifiés. La troisième ligne
indique le nombre de workers réellement exécutés, ceci dépendant de différents
paramètres de configuration (notamment ``max_parallel_workers_per_gather`` et
``max_parallel_workers``).

La quatrième ligne apparaît seulement si les options ``ANALYZE`` et
``BUFFERS`` sont utilisées. Elle indique comment les caches partagé, local et
les fichiers temporaire sont gérés : combien de blocs sont lus dans le cache
et en dehors, modifiés, écrits sur disque, etc.

La cinquième ligne indique un ``Parallel Seq Scan`` mais cela pourrait être
tout autre nœud parallélisé.

Voici un exemple complet pour montrer le calcul du coût::

   Gather  (cost=21898.96..17829247.23 rows=883844 width=28)
     Workers Planned: 4
     ->  Parallel Bitmap Heap Scan on ...  (cost=20898.96..17739862.83 rows=220961 width=28)
           ->  Bitmap Index Scan ...

Le coût de départ dépend de deux variables :

* le coût de départ du nœud suivant (ici le ``Parallel Bitmap Heap Scan``) ;
* et la configuration actuelle du paramètre ``parallel_setup_cost``.

Le coût final dépend de plusieurs variables :

* le coût de départ et le coût final du nœud suivant (ici le ``Parallel Bitmap
  Heap Scan``) ;
* le nombre de lignes estimées ;
* le coût de départ de ce nœud (``Gather``) ;
* et la configuration actuelle du paramètre ``parallel_tuple_cost``.

La formule complète est la suivante::

  cout_total_noeud_suivant - cout_initial_noeud_suivant
  + parallel_tuple_cost * nb_lignes
  + cout_initial_du_noeud;

Sur l'exemple, cela nous donne (pour une configuration par défaut, donc un
`parallel_setup_cost` à 1000 et un `parallel_tuple_cost` à 0,1)::

  17739862.83 − 20898.96 + 0.1×883844 + 21898.96 = 17829247.23

