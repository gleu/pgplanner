Nœuds de parcours de table
==========================

Il existe deux types de nœuds pour les parcours de table.

Le premier est le ``Seq Scan``, un parcours séquentiel, par défaut complet,
de la table. Les données peuvent être filtrées mais elles ne sont pas triées
en sortie de ce type de nœud.

Depuis la version 9.5, un deuxième type de parcours de table est apparu. Il
s'appelle ``Sample Scan`` et permet de récupérer un échantillon d'une table.

.. toctree::
   :maxdepth: 1

   table_scans/seq_scan.rst
   table_scans/sample_scan.rst
