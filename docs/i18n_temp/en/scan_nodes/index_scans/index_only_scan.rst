Index Only Scan
===============

We said earlier that during an index scan, the table is also scanned, at least
a very small part of it. It's useful to get visibility informations, and the
values of other columns.

In some simple queries, the executor doesn't need to know other columns'
values. Moreover, since the 9.2 release, there's a file called the visibility
map, which contains informations about visibility. All this means that,
sometimes, there's no need to scan the table while doing an index scan. When
the planner thinks it is a good idea, it uses another node called ``Index Only
Scan``. For example, on this query::

  SELECT c1 FROM t1 WHERE c1<10;

The index will be able to filter the rows very efficiently, and will have
enough informations to output the values it needs. Hence this new plan::

   EXPLAIN SELECT c1 FROM t1 WHERE c1<10;

                                  QUERY PLAN                                
   -------------------------------------------------------------------------
    Index Only Scan using t1_c1_idx on t1  (cost=0.29..4.45 rows=9 width=4)
      Index Cond: (c1 < 10)
   (2 rows)

So, here is the informations we get from an ``EXPLAIN`` query::

   Index Only Scan using t1_id_idx on t1 ...
     Index Cond: (id < 10)
     Heap Fetches: 0
     Buffers: shared hit=4

The first line gives us the node name, the index name, and the relation name.
The executor will read this index, following the tree structure on disk. At
each value found, it will check if the block the row belongs to is marked
``all visible`` in the visibility map. If the block is marked ``all visible``,
the executor knows the row is visible, and will output the column value.  If
it isn't, it will read the associated table's row to get the visibility
information.

Line #2 appears only when the index can satisfy a predicate. In this example,
the executor uses the index to quickly find the rows for which c1 is less than
1000.

Line #3 only appears if the ``ANALYZE`` option is used with the ``EXPLAIN``
statement.  It specifies how many pages were read in the table, if any.

Line #4 only appears if the ``ANALYZE`` and ``BUFFERS`` options are used. It
says how shared buffers, local buffers and temporary buffers are handled: how
many pages are read in cache and out of cache, written, flushed, etc.

Let's try on an example. Let's read t1 and get all rows for which c1 is bigger
than 90000::

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

The ``Heap Fetches`` tells us that the executor didn't need to read pages from
the table. Now let's add some rows to the table, and try thet ``EXPLAIN``
again::

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

The insertion added some pages to the table, and there was no ``VACUUM`` in
between, meaning the visibility map hasn't been updated. So, now, the ``Heap
Fetches`` says it had to read 20.1k rows in the table to determine their
visibility status.

Now let's do a ``VACUUM`` and try again::

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

``Heap Fetches`` is back to zero because the visibility map knew all rows were
visible in all the interesting pages.

B-Tree indexes were the first to support index only scans. GiST and SP-GiST
indexes are compatible since the 9.5 release, but only with some operator
classes. GIN indexes don't support index only scan. BRIN indexes won't ever
support them because they don't contain the values, only ranges of values.
Partial indexes support this type of scan since the 9.6 release.

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
