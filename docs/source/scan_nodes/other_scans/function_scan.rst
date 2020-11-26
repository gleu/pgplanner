Function Scan
=============

Une fonction appelée dans la clause ``FROM`` ou dans une jointure utilise un
nœud ``Function Scan``.

En voici un exemple ::

   Function Scan on f1 (cost=0.25..10.25 rows=1000 width=4)

Par défaut, une fonction SRF (c'est-à-dire pouvant renvoyer plusieurs lignes)
écrite en PL a un coût de 100. Une fonction SRF écrite en C a par défaut un
coût de 1. L'estimation du nombre de lignes renvoyés par toute fonction SRF
est de 1000.

Vous pouvez configurer ces deux informations (coût et nombre de lignes) lors
de la création ou de la modification d'une fonction. Vous avez besoin pour
cela des clauses ``ROWS`` et ``COST`` pour les configurer statiquement. Le
coût a une unité correspondant à ``cpu_operator_cost``, ceci signifiant que
vous aurez à multiplier le coût de la fonction avec la valeur du paramètre
``cpu_operator_cost`` et avec le nombre de lignes pour obtenir le coût du
filtre. De ce fait, la requête pour le coût final devient ::

   SELECT
     round((
       current_setting('seq_page_cost')::numeric*relpages +
       current_setting('cpu_tuple_cost')::numeric*reltuples +
       <FUNCTION_COST>*current_setting('cpu_operator_cost')::numeric*reltuples
     )::numeric, 2)
   AS final_cost
   FROM pg_class
   WHERE relname='t1';

Depuis la version 12, vous pouvez configurer les deux informations
dynamiquement grâce aux fonctions de support.  Voir
https://www.cybertec-postgresql.com/en/optimizer-support-functions/ pour une
belle introduction sur ce sujet.

Une fonction standard, renvoyant une ligne, a bien sûr un nombre de lignes de
1 et un coût par défaut de 0,01.

