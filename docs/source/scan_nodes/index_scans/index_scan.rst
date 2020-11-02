Index Scan
==========

The Index Scan is the oldest way of scanning an index. It's still relevant
though, but there are some optimized ways to scan an index.

When the executor does an Index Scan, it will scan the index to find the
values it needs. Each time it finds a good value, it will read the row in the
table. It needs to do this because only the table row contains visibility
information (the index doesn't). It may also need other columns' value to
answer the query.

Here are the informations the ``EXPLAIN`` statement gives us::

   Index Scan using t1_id_idx on t1 ...
     Index Cond: (c1 < 1000)
     Filter: (c2 = 'Line 30')
     Rows Removed by Filter: 66
     Buffers: shared hit=6

The first line gives us the node name, the index name, and the relation name.
The executor will read this index, following the tree structure on disk. At
each value found, it will read the associated table's row.

Line #2 appears only when the index can satisfy a predicate In this example,
the executor uses the index to quickly find the rows for which c1 is less than
1000.

Line #3 may appear if there's another filter used on the index' relation. This
filtering happens once the associated row on the table has been read.

Line #4 only appears when the executor needs to filter data other than with
the index and if the ``ANALYZE`` option is used with the ``EXPLAIN``
statement.  It says how many rows were filtered out by applying the predicate.

Line #5 only appears if the ``ANALYZE`` and ``BUFFERS`` options are used. It
says how shared buffers, local buffers and temporary buffers are handled: how
many pages are read in cache and out of cache, written, flushed, etc.

So, if we had an index to our table ``t1``, here is an index scan on the
table::

   CREATE INDEX ON t1(c1);
   EXPLAIN SELECT * FROM t1 WHERE c1=1000;

                                QUERY PLAN
   ---------------------------------------------------------------------
    Index Scan using t1_c1_idx on t1  (cost=0.29..8.31 rows=1 width=14)
      Index Cond: (c1 = 1000)
   (2 rows)

An Index Scan index can also be used to answer an ``ORDER BY`` clause in a
query, though the index should be a B-Tree. Data in a B-Tree index is already
ordered, so the executor just needs to follow the order::

   EXPLAIN SELECT * FROM t1 ORDER BY c1;

                                    QUERY PLAN
   -----------------------------------------------------------------------------
    Index Scan using t1_c1_idx on t1  (cost=0.29..3148.29 rows=100000 width=14)
   (1 row)

If the executor needs the biggest values first, it can do a reverse or
backward scan::

   EXPLAIN SELECT * FROM t1 ORDER BY c1 DESC;

                                         QUERY PLAN
   --------------------------------------------------------------------------------------
    Index Scan Backward using t1_c1_idx on t1  (cost=0.29..3148.29 rows=100000 width=14)
   (1 row)

These two only work on B-Tree indexes. For now, the other index access methods
can't be used for this.

Due to the tree structure of an index, doing an index scan is pretty slow
because the OS needs to move the heads of the disk in order to find the next
interesting page. This behaviour tends to disable the Read Ahead functionality
of the OS. Because of this, reading the same amount of data in a table and in
an index will have completely different performances. The index scan will be
longer, unless the index is available in memory or on a SSD disk. In both
latter cases, there's no need to move the disk's heads and therefore, the scan
is faster.

The ``random_page_cost`` is the cost associated with reading a page in an
index. This cost is higher than the ``seq_page_cost`` because of the need to
move the head. By default, the ``random_page_cost`` is four times higher than
``seq_page_cost``. Lowering ``random_page_cost`` will help the planner to
choose an index scan because that will lower the total cost. Here is an
example with two different values for this GUC::

   SET random_page_cost TO 4;
   EXPLAIN SELECT * FROM t1 ORDER BY c1 DESC;

                                         QUERY PLAN
   --------------------------------------------------------------------------------------
    Index Scan Backward using t1_c1_idx on t1  (cost=0.29..3148.29 rows=100000 width=14)
   (1 row)

   SET random_page_cost TO 1;
   EXPLAIN SELECT * FROM t1 ORDER BY c1 DESC;

                                         QUERY PLAN
   --------------------------------------------------------------------------------------
    Index Scan Backward using t1_c1_idx on t1  (cost=0.29..2317.29 rows=100000 width=14)
   (1 row)

By lowering ``random_page_cost`` from 4 to 1, the total cost is actually quite
lower too.

The ``enable_indexscan`` GUC allows us to enable or disable index scans. You
shouldn't disable it globally.
