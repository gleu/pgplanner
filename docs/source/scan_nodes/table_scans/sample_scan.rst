Sample Scan
===========

You may need to read only a sample of a table. The SQL standard allows you to
do this with the TABLESAMPLE clause. PostgreSQL supports this clause, and
allows you to specifiy the sampling method.

Here is an example of such a query::

   SELECT * FROM t1 TABLESAMPLE SYSTEM(10)
   WHERE id=10;

And here are the informations the ``EXPLAIN`` statement gives us::

   Sample Scan on t1 [...]
     Sampling: system ('10'::real)
     Filter: (id = 10)
     Rows Removed by Filter: 999999
     Buffers: shared hit=4005 read=420

The first line is the node name, followed by the relation name. The executor
will read this table, pages by pages, read each tuple, and keep only those
that are visible by the current transaction and that respect the filter (if
there's one).

Line #2 specifies the sampling method.

Line #3 only appears when the executor needs to filter data because of a
predicate (a ``WHERE`` clause for example). It says which filter is done
(here, it needs to get all the rows in which the value of id is equal to 10).

Line #4 only appears when the executor needs to filter data because of a
predicate and if the ``ANALYZE`` option is used with the ``EXPLAIN``
statement.  It says how many rows were filtered out by applying the predicate.
If this number is a lot more then the number of rows returned by the node, an
index might help to lower the execution time of the query.

Line #5 only appears if the ``ANALYZE`` and ``BUFFERS`` options are used. It
says how shared buffers, local buffers and temporary buffers are handled: how
many pages are read in cache and out of cache, written, flushed, etc.

