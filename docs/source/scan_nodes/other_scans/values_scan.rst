Values Scan
===========

L'instruction ``VALUES`` a son propre nœud, appelé ``Values``. Il ne donne pas
d'informations spéciales, en dehors des habituelles. Par exemple ::

   Values Scan on "*VALUES*" (cost=0.00..0.04 rows=3 width=4)

Le nombre de lignes devrait être exact.

La formule du coût ressemble principalement à ceci ::

   (cpu_operator_cost + cpu_tuple_cost) * baserel->tuples

Par exemple ::

   EXPLAIN VALUES (1), (2), (3), (4), (5), (6);

                            QUERY PLAN                          
   -------------------------------------------------------------
    Values Scan on "*VALUES*"  (cost=0.00..0.08 rows=6 width=4)
   (1 row)
   
   SELECT round(
           (  current_setting('cpu_operator_cost')::numeric
            + current_setting('cpu_tuple_cost')::numeric)
           * 6,
           2);
    round 
   -------
     0.08
   (1 row)

Pour la valeur du champ ``width``, l'optimiseur ne peut pas utiliser les
statistiques. Sur les valeurs à largeur fixe, c'est simple ::

   EXPLAIN VALUES (1), (2), (3), (4), (5), (6);

                            QUERY PLAN
   -------------------------------------------------------------
    Values Scan on "*VALUES*"  (cost=0.00..0.08 rows=6 width=4)
   (1 row)

Pour une donnée de taille variable, c'est beaucoup moins simple.

Pour un champ char(X), X est le nombre de caractère. Il faut multiplier ce
nombre par la taille max d'un caractère pour l'encodage de la base (4 octets
pour UTF8 par exemple), et il faut ajouter l'en-tête de 4 octets. Donc pour un
char(100), on se trouve avec une taille max de 404 octets. Pour un char(X), la
taille max se trouve être la seule largeur. Autrement dit::

   EXPLAIN VALUES ('1'::char(200)),
                  ('2'::char(200)),
                  ('3'::char(200)),
                  ('4'::char(200)),
                  ('5'::char(200)),
                  ('01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567891'::char(200));

                             QUERY PLAN
   ---------------------------------------------------------------
    Values Scan on "*VALUES*"  (cost=0.00..0.08 rows=6 width=804)
   (1 row)

Pour un type ``varchar``, le calcul du nombre d'octets est identique (donc
4+X*taille_encodage). Cependant, la largeur maximale ne correspond pas
forcément à la largeur moyenne. On suit donc l'heuristique suivante::

   if (maxwidth <= 32)
       return maxwidth;    /* assume full width */
   if (maxwidth < 1000)
       return 32 + (maxwidth - 32) / 2;    /* assume 50% */
   return 32 + (1000 - 32) / 2; 

De ce fait::

   EXPLAIN VALUES ('1'::varchar(200)),
                  ('2'::varchar(200)),
                  ('3'::varchar(200)),
                  ('4'::varchar(200)),
                  ('5'::varchar(200)),
                  ('01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567891'::varchar(200));

                             QUERY PLAN
   ---------------------------------------------------------------
    Values Scan on "*VALUES*"  (cost=0.00..0.08 rows=6 width=418)
   (1 row)

Enfin, quand le type est inconnu, on utilise une valeur fixe de 32 ::

   EXPLAIN VALUES ('1'), ('2'), ('3'), ('4'), ('5'),
                  ('01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567891');

                             QUERY PLAN
   --------------------------------------------------------------
    Values Scan on "*VALUES*"  (cost=0.00..0.08 rows=6 width=32)
   (1 row)

