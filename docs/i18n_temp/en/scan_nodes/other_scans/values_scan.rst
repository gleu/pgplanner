Values Scan
===========

L'instruction VALUES fait appel à un type de nœuds appelé Values
Values Scan on "*VALUES*"
Scan .
(cost=0.00..0.04 rows=3 width=4)
Le nombre de lignes est toujours exact. Le coût correspond à
nombre_lignes .

(cpu_operator_cost + cpu_tuple_cost + qpqual_cost.per_tuple) * baserel->tuples






4350 static void
4351 get_restriction_qual_cost(PlannerInfo *root, RelOptInfo *baserel,
4352                           ParamPathInfo *param_info,
4353                           QualCost *qpqual_cost)
4354 {
4355     if (param_info)
4356     {
4357         /* Include costs of pushed-down clauses */
4358         cost_qual_eval(qpqual_cost, param_info->ppi_clauses, root);
4359
4360         qpqual_cost->startup += baserel->baserestrictcost.startup;
4361         qpqual_cost->per_tuple += baserel->baserestrictcost.per_tuple;
4362     }
4363     else
4364         *qpqual_cost = baserel->baserestrictcost;
4365 }

1991 /*
1992  * create_valuesscan_path
1993  *    Creates a path corresponding to a scan of a VALUES list,
1994  *    returning the pathnode.
1995  */
1996 Path *
1997 create_valuesscan_path(PlannerInfo *root, RelOptInfo *rel,
1998                        Relids required_outer)
1999 {
2000     Path       *pathnode = makeNode(Path);
2001
2002     pathnode->pathtype = T_ValuesScan;
2003     pathnode->parent = rel;
2004     pathnode->pathtarget = rel->reltarget;
2005     pathnode->param_info = get_baserel_parampathinfo(root, rel,
2006                                                      required_outer);
2007     pathnode->parallel_aware = false;
2008     pathnode->parallel_safe = rel->consider_parallel;
2009     pathnode->parallel_workers = 0;
2010     pathnode->pathkeys = NIL;   /* result is always unordered */
2011
2012     cost_valuesscan(pathnode, root, rel, pathnode->param_info);
2013
2014     return pathnode;
2015 }
:wq




set_values_size_estimates
