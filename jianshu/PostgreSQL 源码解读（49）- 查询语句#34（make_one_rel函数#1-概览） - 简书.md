先前的章节已介绍了函数query_planner中子函数create_lateral_join_info、match_foreign_keys_to_quals和extract_restriction_or_clauses的主要实现逻辑,本节介绍query_planner中主计划函数make_one_rel的主实现逻辑。

query_planner代码片段:

    
    
         //...
    
         /*
          * Ready to do the primary planning.
          */
         final_rel = make_one_rel(root, joinlist);//执行主要的计划过程
     
         /* Check that we got at least one usable path */
         if (!final_rel || !final_rel->cheapest_total_path ||
             final_rel->cheapest_total_path->param_info != NULL)
             elog(ERROR, "failed to construct the join relation");//检查
     
         return final_rel;//返回结果
    
         //...
    

### 一、数据结构

**RelOptInfo**

    
    
     typedef struct RelOptInfo
     {
         NodeTag     type;//节点标识
     
         RelOptKind  reloptkind;//RelOpt类型
     
         /* all relations included in this RelOptInfo */
         Relids      relids;         /*Relids(rtindex)集合 set of base relids (rangetable indexes) */
     
         /* size estimates generated by planner */
         double      rows;           /*结果元组的估算数量 estimated number of result tuples */
     
         /* per-relation planner control flags */
         bool        consider_startup;   /*是否考虑启动成本?是,需要保留启动成本低的路径 keep cheap-startup-cost paths? */
         bool        consider_param_startup; /*是否考虑参数化?的路径 ditto, for parameterized paths? */
         bool        consider_parallel;  /*是否考虑并行处理路径 consider parallel paths? */
     
         /* default result targetlist for Paths scanning this relation */
         struct PathTarget *reltarget;   /*扫描该Relation时默认的结果 list of Vars/Exprs, cost, width */
     
         /* materialization information */
         List       *pathlist;       /*访问路径链表 Path structures */
         List       *ppilist;        /*路径链表中使用参数化路径进行 ParamPathInfos used in pathlist */
         List       *partial_pathlist;   /* partial Paths */
         struct Path *cheapest_startup_path;//代价最低的启动路径
         struct Path *cheapest_total_path;//代价最低的整体路径
         struct Path *cheapest_unique_path;//代价最低的获取唯一值的路径
         List       *cheapest_parameterized_paths;//代价最低的参数化?路径链表
     
         /* parameterization information needed for both base rels and join rels */
         /* (see also lateral_vars and lateral_referencers) */
         Relids      direct_lateral_relids;  /*使用lateral语法,需依赖的Relids rels directly laterally referenced */
         Relids      lateral_relids; /* minimum parameterization of rel */
     
         /* information about a base rel (not set for join rels!) */
         //reloptkind=RELOPT_BASEREL时使用的数据结构
         Index       relid;          /* Relation ID */
         Oid         reltablespace;  /* 表空间 containing tablespace */
         RTEKind     rtekind;        /* 基表?子查询?还是函数等等?RELATION, SUBQUERY, FUNCTION, etc */
         AttrNumber  min_attr;       /* 最小的属性编号 smallest attrno of rel (often <0) */
         AttrNumber  max_attr;       /* 最大的属性编号 largest attrno of rel */
         Relids     *attr_needed;    /* 数组 array indexed [min_attr .. max_attr] */
         int32      *attr_widths;    /* 属性宽度 array indexed [min_attr .. max_attr] */
         List       *lateral_vars;   /* 关系依赖的Vars/PHVs LATERAL Vars and PHVs referenced by rel */
         Relids      lateral_referencers;    /*依赖该关系的Relids rels that reference me laterally */
         List       *indexlist;      /* 该关系的IndexOptInfo链表 list of IndexOptInfo */
         List       *statlist;       /* 统计信息链表 list of StatisticExtInfo */
         BlockNumber pages;          /* 块数 size estimates derived from pg_class */
         double      tuples;         /* 元组数 */
         double      allvisfrac;     /* ? */
         PlannerInfo *subroot;       /* 如为子查询,存储子查询的root if subquery */
         List       *subplan_params; /* 如为子查询,存储子查询的参数 if subquery */
         int         rel_parallel_workers;   /* 并行执行,需要多少个workers? wanted number of parallel workers */
     
         /* Information about foreign tables and foreign joins */
         //FWD相关信息
         Oid         serverid;       /* identifies server for the table or join */
         Oid         userid;         /* identifies user to check access as */
         bool        useridiscurrent;    /* join is only valid for current user */
         /* use "struct FdwRoutine" to avoid including fdwapi.h here */
         struct FdwRoutine *fdwroutine;
         void       *fdw_private;
     
         /* cache space for remembering if we have proven this relation unique */
         //已知的,可保证唯一的Relids链表
         List       *unique_for_rels;    /* known unique for these other relid
                                          * set(s) */
         List       *non_unique_for_rels;    /* 已知的,不唯一的Relids链表 known not unique for these set(s) */
     
         /* used by various scans and joins: */
         List       *baserestrictinfo;   /* 如为基本关系,存储约束条件 RestrictInfo structures (if base rel) */
         QualCost    baserestrictcost;   /* 解析约束表达式的成本? cost of evaluating the above */
         Index       baserestrict_min_security;  /* 最低安全等级 min security_level found in
                                                  * baserestrictinfo */
         List       *joininfo;       /* 连接语句的约束条件信息 RestrictInfo structures for join clauses
                                      * involving this rel */
         bool        has_eclass_joins;   /* 是否存在等价类连接? T means joininfo is incomplete */
     
         /* used by partitionwise joins: */
         bool        consider_partitionwise_join;    /* 分区? consider partitionwise
                                                      * join paths? (if
                                                      * partitioned rel) */
         Relids      top_parent_relids;  /* Relids of topmost parents (if "other"
                                          * rel) */
     
         /* used for partitioned relations */
         //分区表使用
         PartitionScheme part_scheme;    /* 分区的schema Partitioning scheme. */
         int         nparts;         /* 分区数 number of partitions */
         struct PartitionBoundInfoData *boundinfo;   /* 分区边界信息 Partition bounds */
         List       *partition_qual; /* 分区约束 partition constraint */
         struct RelOptInfo **part_rels;  /* 分区的RelOptInfo数组 Array of RelOptInfos of partitions,
                                          * stored in the same order of bounds */
         List      **partexprs;      /* 非空分区键表达式 Non-nullable partition key expressions. */
         List      **nullable_partexprs; /* 可为空的分区键表达式 Nullable partition key expressions. */
         List       *partitioned_child_rels; /* RT Indexes链表 List of RT indexes. */
     } RelOptInfo;
    
    

### 二、源码解读

**make_one_rel**  
make_one_rel函数找出执行查询的所有可能访问路径,但不考虑上层的Non-SPJ操作,返回一个最上层的RelOptInfo.  
make_one_rel函数分为两个阶段:生成扫描路径(set_base_rel_pathlists)和生成连接路径(make_rel_from_joinlist).  
注:SPJ是指选择(Select)/投影(Project)/连接(Join),相对应的Non-SPJ操作是指Group分组/Sort排序等操作

    
    
     /*
      * make_one_rel
      *    Finds all possible access paths for executing a query, returning a
      *    single rel that represents the join of all base rels in the query.
      */
     RelOptInfo *
     make_one_rel(PlannerInfo *root, List *joinlist)
     {
         RelOptInfo *rel;
         Index       rti;
     
         /*
          * Construct the all_baserels Relids set.
          */
         root->all_baserels = NULL;
         for (rti = 1; rti < root->simple_rel_array_size; rti++)//遍历RelOptInfo
         {
             RelOptInfo *brel = root->simple_rel_array[rti];
     
             /* there may be empty slots corresponding to non-baserel RTEs */
             if (brel == NULL)
                 continue;
     
             Assert(brel->relid == rti); /* sanity check on array */
     
             /* ignore RTEs that are "other rels" */
             if (brel->reloptkind != RELOPT_BASEREL)
                 continue;
     
             root->all_baserels = bms_add_member(root->all_baserels, brel->relid);//添加到all_baserels遍历中
         }
     
         /* Mark base rels as to whether we care about fast-start plans */
         //设置RelOptInfo的consider_param_startup变量,是否考量fast-start plans
         set_base_rel_consider_startup(root);
     
         /*
          * Compute size estimates and consider_parallel flags for each base rel,
          * then generate access paths.
          */
         set_base_rel_sizes(root);//估算Relation的Size并且设置consider_parallel标记
         set_base_rel_pathlists(root);//生成Relation的扫描(访问)路径
     
         /*
          * Generate access paths for the entire join tree.
          * 通过动态规划或遗传算法生成连接路径 
          */
         rel = make_rel_from_joinlist(root, joinlist);
     
         /*
          * The result should join all and only the query's base rels.
          */
         Assert(bms_equal(rel->relids, root->all_baserels));
         //返回最上层的RelOptInfo
         return rel;
     }
    
    //--------------------------------------------------------     /*
      * set_base_rel_consider_startup
      *    Set the consider_[param_]startup flags for each base-relation entry.
      *
      * For the moment, we only deal with consider_param_startup here; because the
      * logic for consider_startup is pretty trivial and is the same for every base
      * relation, we just let build_simple_rel() initialize that flag correctly to
      * start with.  If that logic ever gets more complicated it would probably
      * be better to move it here.
      */
     static void
     set_base_rel_consider_startup(PlannerInfo *root)
     {
         /*
          * Since parameterized paths can only be used on the inside of a nestloop
          * join plan, there is usually little value in considering fast-start
          * plans for them.  However, for relations that are on the RHS of a SEMI
          * or ANTI join, a fast-start plan can be useful because we're only going
          * to care about fetching one tuple anyway.
          *
          * To minimize growth of planning time, we currently restrict this to
          * cases where the RHS is a single base relation, not a join; there is no
          * provision for consider_param_startup to get set at all on joinrels.
          * Also we don't worry about appendrels.  costsize.c's costing rules for
          * nestloop semi/antijoins don't consider such cases either.
          */
         ListCell   *lc;
     
         foreach(lc, root->join_info_list)
         {
             SpecialJoinInfo *sjinfo = (SpecialJoinInfo *) lfirst(lc);
             int         varno;
     
             if ((sjinfo->jointype == JOIN_SEMI || sjinfo->jointype == JOIN_ANTI) &&
                 bms_get_singleton_member(sjinfo->syn_righthand, &varno))
             {
                 RelOptInfo *rel = find_base_rel(root, varno);
     
                 rel->consider_param_startup = true;
             }
         }
     }
    
    //--------------------------------------------------------     /*
      * set_base_rel_sizes
      *    Set the size estimates (rows and widths) for each base-relation entry.
      *    Also determine whether to consider parallel paths for base relations.
      *
      * We do this in a separate pass over the base rels so that rowcount
      * estimates are available for parameterized path generation, and also so
      * that each rel's consider_parallel flag is set correctly before we begin to
      * generate paths.
      */
     static void
     set_base_rel_sizes(PlannerInfo *root)
     {
         Index       rti;
     
         for (rti = 1; rti < root->simple_rel_array_size; rti++)//遍历RelOptInfo数组
         {
             RelOptInfo *rel = root->simple_rel_array[rti];
             RangeTblEntry *rte;
     
             /* there may be empty slots corresponding to non-baserel RTEs */
             if (rel == NULL)
                 continue;
     
             Assert(rel->relid == rti);  /* sanity check on array */
     
             /* ignore RTEs that are "other rels" */
             if (rel->reloptkind != RELOPT_BASEREL)
                 continue;
     
             rte = root->simple_rte_array[rti];
     
             /*
              * If parallelism is allowable for this query in general, see whether
              * it's allowable for this rel in particular.  We have to do this
              * before set_rel_size(), because (a) if this rel is an inheritance
              * parent, set_append_rel_size() will use and perhaps change the rel's
              * consider_parallel flag, and (b) for some RTE types, set_rel_size()
              * goes ahead and makes paths immediately.
              */
             if (root->glob->parallelModeOK)
                 set_rel_consider_parallel(root, rel, rte);
     
             set_rel_size(root, rel, rti, rte);
         }
     }
    
     /*
      * set_rel_size
      *    Set size estimates for a base relation
      */
     static void
     set_rel_size(PlannerInfo *root, RelOptInfo *rel,
                  Index rti, RangeTblEntry *rte)
     {
         if (rel->reloptkind == RELOPT_BASEREL &&
             relation_excluded_by_constraints(root, rel, rte))
         {
             /*
              * We proved we don't need to scan the rel via constraint exclusion,
              * so set up a single dummy path for it.  Here we only check this for
              * regular baserels; if it's an otherrel, CE was already checked in
              * set_append_rel_size().
              *
              * In this case, we go ahead and set up the relation's path right away
              * instead of leaving it for set_rel_pathlist to do.  This is because
              * we don't have a convention for marking a rel as dummy except by
              * assigning a dummy path to it.
              */
             set_dummy_rel_pathlist(rel);//
         }
         else if (rte->inh)//inherit table
         {
             /* It's an "append relation", process accordingly */
             set_append_rel_size(root, rel, rti, rte);
         }
         else
         {
             switch (rel->rtekind)
             {
                 case RTE_RELATION://数据表
                     if (rte->relkind == RELKIND_FOREIGN_TABLE)//FDW
                     {
                         /* Foreign table */
                         set_foreign_size(root, rel, rte);
                     }
                     else if (rte->relkind == RELKIND_PARTITIONED_TABLE)//分区表
                     {
                         /*
                          * A partitioned table without any partitions is marked as
                          * a dummy rel.
                          */
                         set_dummy_rel_pathlist(rel);
                     }
                     else if (rte->tablesample != NULL)//采样表
                     {
                         /* Sampled relation */
                         set_tablesample_rel_size(root, rel, rte);
                     }
                     else
                     {
                         /* Plain relation */
                         set_plain_rel_size(root, rel, rte);//常规的数据表
                     }
                     break;
                 case RTE_SUBQUERY://子查询
     
                     /*
                      * Subqueries don't support making a choice between
                      * parameterized and unparameterized paths, so just go ahead
                      * and build their paths immediately.
                      */
                     set_subquery_pathlist(root, rel, rti, rte);//生成子查询访问路径
                     break;
                 case RTE_FUNCTION://FUNCTION
                     set_function_size_estimates(root, rel);
                     break;
                 case RTE_TABLEFUNC://TABLEFUNC
                     set_tablefunc_size_estimates(root, rel);
                     break;
                 case RTE_VALUES://VALUES
                     set_values_size_estimates(root, rel);
                     break;
                 case RTE_CTE://CTE
     
                     /*
                      * CTEs don't support making a choice between parameterized
                      * and unparameterized paths, so just go ahead and build their
                      * paths immediately.
                      */
                     if (rte->self_reference)
                         set_worktable_pathlist(root, rel, rte);
                     else
                         set_cte_pathlist(root, rel, rte);
                     break;
                 case RTE_NAMEDTUPLESTORE://NAMEDTUPLESTORE,命名元组存储
                     set_namedtuplestore_pathlist(root, rel, rte);
                     break;
                 default:
                     elog(ERROR, "unexpected rtekind: %d", (int) rel->rtekind);
                     break;
             }
         }
     
         /*
          * We insist that all non-dummy rels have a nonzero rowcount estimate.
          */
         Assert(rel->rows > 0 || IS_DUMMY_REL(rel));
     }
     
    //--------------------------------------------------------     /*
      * set_base_rel_pathlists
      *    Finds all paths available for scanning each base-relation entry.
      *    Sequential scan and any available indices are considered.
      *    Each useful path is attached to its relation's 'pathlist' field.
      *
      *    为每一个base rels找出所有可用的访问路径(顺序扫描和所有可用的索引都会考虑在内)
      *    每一个可用的路径都会添加到pathlist链表中
      *
      */
     static void
     set_base_rel_pathlists(PlannerInfo *root)
     {
         Index       rti;
     
         for (rti = 1; rti < root->simple_rel_array_size; rti++)//遍历RelOptInfo数组
         {
             RelOptInfo *rel = root->simple_rel_array[rti];
     
             /* there may be empty slots corresponding to non-baserel RTEs */
             if (rel == NULL)
                 continue;
     
             Assert(rel->relid == rti);  /* sanity check on array */
     
             /* ignore RTEs that are "other rels" */
             if (rel->reloptkind != RELOPT_BASEREL)
                 continue;
     
             set_rel_pathlist(root, rel, rti, root->simple_rte_array[rti]);
         }
     }
    
     /*
      * set_rel_pathlist
      *    Build access paths for a base relation
      */
     static void
     set_rel_pathlist(PlannerInfo *root, RelOptInfo *rel,
                      Index rti, RangeTblEntry *rte)
     {
         if (IS_DUMMY_REL(rel))
         {
             /* We already proved the relation empty, so nothing more to do */
         }
         else if (rte->inh)//inherit
         {
             /* It's an "append relation", process accordingly */
             set_append_rel_pathlist(root, rel, rti, rte);
         }
         else//常规
         {
             switch (rel->rtekind)
             {
                 case RTE_RELATION://数据表
                     if (rte->relkind == RELKIND_FOREIGN_TABLE)//FDW
                     {
                         /* Foreign table */
                         set_foreign_pathlist(root, rel, rte);
                     }
                     else if (rte->tablesample != NULL)//采样表
                     {
                         /* Sampled relation */
                         set_tablesample_rel_pathlist(root, rel, rte);
                     }
                     else//常规数据表
                     {
                         /* Plain relation */
                         set_plain_rel_pathlist(root, rel, rte);
                     }
                     break;
                 case RTE_SUBQUERY://子查询
                     /* Subquery --- 已在set_rel_size处理,fully handled during set_rel_size */
                     break;
                 case RTE_FUNCTION:
                     /* RangeFunction */
                     set_function_pathlist(root, rel, rte);
                     break;
                 case RTE_TABLEFUNC:
                     /* Table Function */
                     set_tablefunc_pathlist(root, rel, rte);
                     break;
                 case RTE_VALUES:
                     /* Values list */
                     set_values_pathlist(root, rel, rte);
                     break;
                 case RTE_CTE:
                     /* CTE reference --- fully handled during set_rel_size */
                     break;
                 case RTE_NAMEDTUPLESTORE:
                     /* tuplestore reference --- fully handled during set_rel_size */
                     break;
                 default:
                     elog(ERROR, "unexpected rtekind: %d", (int) rel->rtekind);
                     break;
             }
         }
     
         /*
          * If this is a baserel, we should normally consider gathering any partial
          * paths we may have created for it.
          *
          * However, if this is an inheritance child, skip it.  Otherwise, we could
          * end up with a very large number of gather nodes, each trying to grab
          * its own pool of workers.  Instead, we'll consider gathering partial
          * paths for the parent appendrel.
          *
          * Also, if this is the topmost scan/join rel (that is, the only baserel),
          * we postpone this until the final scan/join targelist is available (see
          * grouping_planner).
          */
         if (rel->reloptkind == RELOPT_BASEREL &&
             bms_membership(root->all_baserels) != BMS_SINGLETON)
             generate_gather_paths(root, rel, false);
     
         /*
          * Allow a plugin to editorialize on the set of Paths for this base
          * relation.  It could add new paths (such as CustomPaths) by calling
          * add_path(), or delete or modify paths added by the core code.
          */
         if (set_rel_pathlist_hook)//钩子函数
             (*set_rel_pathlist_hook) (root, rel, rti, rte);
     
         /* Now find the cheapest of the paths for this rel */
         set_cheapest(rel);//找出代价最低的访问路径
     
     #ifdef OPTIMIZER_DEBUG
         debug_print_rel(root, rel);
     #endif
     }
    
    //------------------------------------------------------------     /*
      * make_rel_from_joinlist
      *    Build access paths using a "joinlist" to guide the join path search.
      *
      *    根据joinlist构建连接访问路径,joinlist是函数deconstruct_jointree函数的返回
      *
      * See comments for deconstruct_jointree() for definition of the joinlist
      * data structure.
      */
     static RelOptInfo *
     make_rel_from_joinlist(PlannerInfo *root, List *joinlist)
     {
         int         levels_needed;
         List       *initial_rels;
         ListCell   *jl;
     
         /*
          * Count the number of child joinlist nodes.  This is the depth of the
          * dynamic-programming algorithm we must employ to consider all ways of
          * joining the child nodes.
          */
         levels_needed = list_length(joinlist);//joinlist链表长度
     
         if (levels_needed <= 0)
             return NULL;            /* nothing to do? */
     
         /*
          * Construct a list of rels corresponding to the child joinlist nodes.
          * This may contain both base rels and rels constructed according to
          * sub-joinlists.
          */
         initial_rels = NIL;
         foreach(jl, joinlist)//遍历链表
         {
             Node       *jlnode = (Node *) lfirst(jl);
             RelOptInfo *thisrel;
     
             if (IsA(jlnode, RangeTblRef))//RTR
             {
                 int         varno = ((RangeTblRef *) jlnode)->rtindex;
     
                 thisrel = find_base_rel(root, varno);
             }
             else if (IsA(jlnode, List))
             {
                 /* Recurse to handle subproblem */
                 thisrel = make_rel_from_joinlist(root, (List *) jlnode);//递归调用
             }
             else
             {
                 elog(ERROR, "unrecognized joinlist node type: %d",
                      (int) nodeTag(jlnode));
                 thisrel = NULL;     /* keep compiler quiet */
             }
     
             initial_rels = lappend(initial_rels, thisrel);//加入初始化链表中
         }
     
         if (levels_needed == 1)
         {
             /*
              * Single joinlist node, so we're done.
              */
             return (RelOptInfo *) linitial(initial_rels);
         }
         else
         {
             /*
              * Consider the different orders in which we could join the rels,
              * using a plugin, GEQO, or the regular join search code.
              *
              * We put the initial_rels list into a PlannerInfo field because
              * has_legal_joinclause() needs to look at it (ugly :-().
              */
             root->initial_rels = initial_rels;
     
             if (join_search_hook)//钩子函数
                 return (*join_search_hook) (root, levels_needed, initial_rels);
             else if (enable_geqo && levels_needed >= geqo_threshold)
                 return geqo(root, levels_needed, initial_rels);//通过遗传算法构建连接访问路径
             else
                 return standard_join_search(root, levels_needed, initial_rels);//通过动态规划算法构建连接路径
         }
     }
    
    

### 三、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)

