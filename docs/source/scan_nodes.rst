Nœuds de parcours
=================

Il existe plusieurs types de nœuds de parcours.

Il y a tout d'abord les parcours de table. Ils n'ont pas besoin d'objets
supplémentaires, ils sont toujours utilisables, et c'est certainement leur
plus grand avantage. On parle de parcours de table, mais ils sont aussi
utilisés pour parcourir des vues matérialisées.

Il y a ensuite les parcours d'index. Ils ont été ajoutés pour permettre de
meilleures performances lorsqu'on accède une très petite partie des données
d'une table ou lorsqu'on souhaite trier des données.

Enfin, il existe d'autres types de parcours, comme le parcours de fonctions
dans le cas de fonctions SRF.

.. toctree::
   :maxdepth: 1

   scan_nodes/table_scans.rst
   scan_nodes/index_scans.rst
   scan_nodes/other_scans.rst
