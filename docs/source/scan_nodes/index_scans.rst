Nœuds de parcours d'index
=========================

Les types de parcours d'index sont au nombre de trois. Les ``Index Scan`` sont
les plus anciens. Ils sont les plus intéressants quand le filtre est très
discriminant (autrement dit quand le nombre de lignes résultats est très
faible). Les ``Bitmap Scan`` sont intéressants quand le filtre est moins
discriminant ou quand l'optimiseur souhaite combiner l'utilisation de deux
index ou plus. Enfin, les ``Index Only Scan`` sont une micro-optimisation très
efficace. Ils sont utilisables dans des cas très particuliers où toutes les
colonnes en sortie et du filtre sont enregistrés dans l'index.

.. toctree::
   :maxdepth: 1

   index_scans/index_scan.rst
   index_scans/bitmap_scan.rst
   index_scans/index_only_scan.rst
