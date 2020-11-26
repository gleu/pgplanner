Table Function Scan
===================

A function called in the FROM clause or in a join uses a Function Scan node.

Here is an example::

   Function Scan on f1 (cost=0.25..10.25 rows=1000 width=4)

By default, a PL SRF function's cost is 100. A C SRF function's cost is 1.
Number of returned rows is 1000 for every SRF function.

You can set both informations (cost and number of rows) when you create or
alter the function. You need the ROWS and COST clauses to set them statically.
The cost has a ``cpu_operator_cost`` unit, meaning that you will have to
multiply the cost of the function with the ``cpu_operator_cost`` GUC's value
and with the number of tuples to get the filtering cost. Hence the cost query
becoming::

   SELECT
     round((
       current_setting('seq_page_cost')::numeric*relpages +
       current_setting('cpu_tuple_cost')::numeric*reltuples +
       <FUNCTION_COST>*current_setting('cpu_operator_cost')::numeric*reltuples
     )::numeric, 2)
   AS final_cost
   FROM pg_class
   WHERE relname='t1';

Since release 12, you can set both clauses dynamically with support functions.
See https://www.cybertec-postgresql.com/en/optimizer-support-functions/ for a
nice introduction to this topic.

A standard function, returning one row, number of rows is of course 1, and
cost is 0.01 by default.
