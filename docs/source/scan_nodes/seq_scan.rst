Seq Scan
========

One way of reading datas from a table is to scan the table, pages by pages,
sequentially, to decode each row, to filter them if needed, and to send them
back to the user. Hence the usual term of ``Sequential Scan`` (or ``Seq
Scan``).

This scan always reads all the pages of the table, unless the query has a
``LIMIT`` clause which would allow the executor to stop the scan beforehand.
It may not send back all the rows if a filter (such as a ``WHERE`` clause) is
applied. If much of the rows are filtered out, an index scan might be a better
option. If much of the rows are to be sent back, a sequential scan is probably
best.

One of the great things about sequential scan is that it works every time.
Whether there is an index or not, a sequential scan is always doable. It's
also the most efficient scan if the relation is really tiny and if the number
of result rows is really a huge part of the total number of rows.

As blocks are read in sequential order, rows aren't specifically ordered.
They are read on physical order on disk, which doesn't necessarily mean they
will be sorted according to their insertion time order or any specific order.

Here are the informations the ``EXPLAIN`` statement gives us::

   Seq Scan on t1 [...]
     Filter: (id = 10)
     Rows Removed by Filter: 999999
     Buffers: shared hit=4005 read=420

The first line is the node name, followed by the relation name. The executor
will read this table, pages by pages, read each tuple, and keep only those
that are visible by the current transaction and that respect the filter (if
there's one).

Line #2 only appears when the executor needs to filter data because of a
predicate (a ``WHERE`` clause for example). It says which filter is done
(here, it needs to get all the rows in which the value of id is equal to 10).

Line #3 only appears when the executor needs to filter data because of a
predicate and if the ``ANALYZE`` option is used with the ``EXPLAIN``
statement.  It says how many rows were filtered out by applying the predicate.
If this number is a lot more then the number of rows returned by the node, an
index might help to lower the execution time of the query.

Line #4 only appears if the ``ANALYZE`` and ``BUFFERS`` options are used. It
says how shared buffers, local buffers and temporary buffers are handled: how
many pages are read in cache and out of cache, written, flushed, etc.

Quand PostgreSQL ne dispose pas de statistiques sur le nombre de lignes et de
blocs (par exemple, une table vide immédiatement après sa création), l'optimiseur
part sur le principe qu'elle contient dix blocs, et estime le nombre de lignes pour
ces dix blocs. Donc l'estimation du coût final sera différente entre deux tables dont
la taille moyenne des lignes diffère.

Here is how to create the table we will play with::

   CREATE TABLE t1 (c1 integer, c2 text);
   INSERT INTO t1 SELECT i, 'Line '||i FROM generate_series(1, 100000) i;
   ANALYZE t1;

Let's read the whole table. A sequential scan will be faster because it reads
pages sequentially. Moreover, it's the only option since the table doesn't
have any indexes::

   EXPLAIN SELECT * FROM t1;
   
                           QUERY PLAN                        
   -----------------------------------------------------------
    Seq Scan on t1  (cost=0.00..1541.00 rows=100000 width=14)
   (1 row)

The cost depends on different things:

* the number of pages,
* the number of rows,
* and the current setting of the ``seq_page_cost`` and ``cpu_tuple_cost``
  GUCs.

We can simply get it with this query::

   SELECT
     round((
       current_setting('seq_page_cost')::numeric*relpages +
       current_setting('cpu_tuple_cost')::numeric*reltuples
     )::numeric, 2)
   AS final_cost
   FROM pg_class
   WHERE relname='t1';

Executing this query gets us a ``1541.00`` result, which is indeed the final
cost.

If there's a filter, this will cost more because of the work needed to filter
all rows in the table::

   EXPLAIN SELECT * FROM t1 WHERE c1=1000;
   
                        QUERY PLAN
   ------------------------------------------------------
    Seq Scan on t1  (cost=0.00..1791.00 rows=1 width=14)
      Filter: (c1 = 1000)
   (2 rows)

The ``Filter`` row shows which filter is done. The ``rows=`` information tells
us how many rows the planner thinks it will get back once the filter is
applied.

This cost is bigger. We went from 1541 to 1791. The added cost depends on the
number of rows in the table and on the current setting of the
``cpu_operator_cost`` GUC. The cost query becomes::

   SELECT
     round((
       current_setting('seq_page_cost')::numeric*relpages +
       current_setting('cpu_tuple_cost')::numeric*reltuples +
       current_setting('cpu_operator_cost')::numeric*reltuples
     )::numeric, 2)
   AS final_cost
   FROM pg_class
   WHERE relname='t1';

which gives us a final cost of 1791.

If the operator is actually a user function, it may depend on the value of
function ``COST`` clause. The example above shows a cost of 1693 when we use
the equality operator. If we write a PL/pgsql function that uses this
operator, but has a 10k cost, it will result in this::

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

So the planner expects to get 33333 rows after applying the "egal(c1, 1000)"
filter. To know how much rows were actually removed by the filter, we need to
execute the query, which means using the ``EXPLAIN`` option::

  EXPLAIN (ANALYZE, BUFFERD)
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

The cost has definitely exploded because of the ``COST`` set by the function.

The ``enable_seqscan`` GUC allows us to enable or disable sequential scans. It
doesn't stricly disable sequential scans (because there's no other way to scan
a table if there's no index on this table). It simply adds 10:sub:`10` to the
cost, so that we'll get a sequential way only of there's no way to do
something else::

   SET enable_seqscan TO off;
   EXPLAIN SELECT * FROM t1 WHERE equal(c1, 1000);
                                   QUERY PLAN
   --------------------------------------------------------------------------
    Seq Scan on t1  (cost=10000000000.00..10002501541.00 rows=33333 width=4)
      Filter: equal(c1, 1000)
   (2 rows)
   RESET enable_seqscan;

