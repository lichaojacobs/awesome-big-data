---
title: kylin query原理剖析
date: 2017-10-31 11:15:14
tags:
	- kylin
    - Java
    - 源码
---


## 前言

最近我们组负责数据建模的同学抱怨kylin的relization选择策略：同一个project下一条查询语句本来期望命中某一个cube的，结果系统却选择了其他cube。之前也有大概翻阅过kylin这块的实现源码，知道如果同一个project下如果有多个满足条件的的实现，会按照成本排序并选择成本最低的那个实现。对于成本这块的度量标准，没有做过多研究，于是带着问题，对这块源码进行了一次梳理。

## 源码剖析

为使博文简洁相关实现只贴部分核心代码，以下所指的Realization对应于构建好的Cube。

#### 查询入口

- QueryService.doQueryWithCache()

```
  //kylin.query.cache-enabled是否开启，如果开启将会从cache里面去读结果
  if (queryCacheEnabled) {
		sqlResponse = searchQueryInCache(sqlRequest);
  }

  try {
	if (null == sqlResponse) {
		if (isSelect) {
			//查询入口
			sqlResponse = query(sqlRequest);
		} else if (kylinConfig.isPushDownEnabled() && kylinConfig.isPushDownUpdateEnabled()) {
			//如果开启了pushDown的话允许非查询的sql，如update
			sqlResponse = update(sqlRequest);
		} else {	
			logger.debug("Directly return exception as the sql is unsupported, and query pushdown is disabled");
                        throw new BadRequestException(msg.getNOT_SUPPORTED_SQL());
				}
	 ...
  catch(){
  	...
  }

```

这里，我们忽略从缓存中查找（searchQueryInCache），以及非select查询的情况，单单从一次正常的查询进行分析，进入query方法。

- QueryService.query()

query方法相对来说比较简单，记录了query开始和结束的信息，相当于做了一个切面的工作

```
    public SQLResponse query(SQLRequest sqlRequest) throws Exception {
        SQLResponse ret = null;
        try {
            final String user = SecurityContextHolder.getContext().getAuthentication().getName();
            badQueryDetector.queryStart(Thread.currentThread(), sqlRequest, user);

            ret = queryWithSqlMassage(sqlRequest);
            return ret;

        } finally {
            String badReason = (ret != null && ret.isPushDown()) ? BadQueryEntry.ADJ_PUSHDOWN : null;
            badQueryDetector.queryEnd(Thread.currentThread(), badReason);
        }
    }

```

其中badQueryDetector是一个单起的线程，用来统计和监测bad query的。当有bad query时notify相关的观察者，做一些操作，如打印日志，记录bad query等。kylin 中很多事件的通知都是通过生产者消费者模式订阅发布的。继续进入queryWithSqlMessage()

- QueryService.queryWithSqlMessage()

```

Connection conn = null;
try {
	 conn = QueryConnection.getConnection(sqlRequest.getProject());
	 ...
	 return execute(correctedSql, sqlRequest, conn);
	 ...
   } finally {
		DBUtils.closeQuietly(conn);
   }

```

这个方法里首先获取了数据库连接，kylin的查询的中间层是基于Calcite的，接下来会看一下QueryConnection背后的逻辑。不过话说回来kylin这种整个大块的try catch异常捕获的机制某种意义上来说是种不负责任的表现。

- QueryConnection.getConnection():

```
    public static Connection getConnection(String project) throws SQLException {
        if (!isRegister) {
            DriverManager.registerDriver(new Driver());
            isRegister = true;
        }
        File olapTmp = OLAPSchemaFactory.createTempOLAPJson(project, KylinConfig.getInstanceFromEnv());
        Properties info = new Properties();
        info.put("model", olapTmp.getAbsolutePath());
        return DriverManager.getConnection("jdbc:calcite:", info);
    }

```

方法比较简单，主要是通过OLAPSchemaFactory.createTempOLAPJson()生成了连接的元数据文件，用来创建连接


- OLAPSchemaFactory

OLAPSchemaFactory 实现了calcite的 SchemaFactory接口，实现了create方法，用来创建连接时生成Schema

```
    @Override
    public Schema create(SchemaPlus parentSchema, String schemaName, Map<String, Object> operand) {
        String project = (String) operand.get(SCHEMA_PROJECT);
        Schema newSchema = new OLAPSchema(project, schemaName, false);
        return newSchema;
    }
    
    

```

在OLAPSchema的init方法中调用了KylinConfigBase.getStorageUrl方法，此方法返回了我们在配置文件中配置的kylin数据的存储信息

```
public StorageURL getStorageUrl() {
        String url = getOptional("kylin.storage.url", "default@hbase");
        
        // for backward compatibility
        // 对2.0早期版本的配置做了兼容
        if ("hbase".equals(url))
            url = "default@hbase";

        return StorageURL.valueOf(url);
}

```
这里也可以看出kylin默认的存储系统是HBase

- QueryService.execute()

从之前的QueryService.queryWithSqlMessage()方法继续往下深入到 execute()方法

```
ResultSet resultSet = null;
if (isPrepareStatementWithParams(sqlRequest)) {
	stat = conn.prepareStatement(correctedSql); // to be closed in the finally
	PreparedStatement prepared = (PreparedStatement) stat;
	processStatementAttr(prepared, sqlRequest);
	for (int i = 0; i < ((PrepareSqlRequest) sqlRequest).getParams().length; i++) {
			setParam(prepared, i + 1, ((PrepareSqlRequest) sqlRequest).getParams()[i]);
	}
	resultSet = prepared.executeQuery();
} else {
	stat = conn.createStatement();
	processStatementAttr(stat, sqlRequest);
	resultSet = stat.executeQuery(correctedSql);
}

```

- OLAPTable

最后查出的结果是在resultSet里，追踪到这一步发现再往下追踪都是Calcite底层的逻辑了，kylin肯定是对Calcite 做了一定的扩展，并且将结果按照kylin预定义的规则做了各种聚合操作。Calcite文档中表示，可以实现三种类型的Table:

- a simple implementation of Table, using the ScannableTable interface, that enumerates all rows directly;
- a more advanced implementation that implements FilterableTable, and can filter out rows according to simple predicates;
- advanced implementation of Table, using TranslatableTable, that translates to relational operators using planner rules.

发现在core-query模块中OLAPTable 实现了TranslatableTable。而OLAPTable 中实现的asQueryable方法有三种Enumerator的实现，这里默认选的是OLAP的实现。

```

public <T> Queryable<T> asQueryable(QueryProvider queryProvider, SchemaPlus schema, String tableName) {
       return new AbstractTableQueryable<T>(queryProvider, schema, this, tableName) {
            @SuppressWarnings("unchecked")
            public Enumerator<T> enumerator() {
                final OLAPQuery query = new OLAPQuery(EnumeratorTypeEnum.OLAP, 0);
                return (Enumerator<T>) query.enumerator();
            }
        };
    }
    
    
OLAPQuery.enumerator
public Enumerator<Object[]> enumerator() {
        OLAPContext olapContext = OLAPContext.getThreadLocalContextById(contextId);
        switch (type) {
        case OLAP:
            return BackdoorToggles.getPrepareOnly() ? new EmptyEnumerator() : new OLAPEnumerator(olapContext, optiqContext);
        case LOOKUP_TABLE:
            return BackdoorToggles.getPrepareOnly() ? new EmptyEnumerator() : new LookupTableEnumerator(olapContext);
        case HIVE:
            return BackdoorToggles.getPrepareOnly() ? new EmptyEnumerator() : new HiveEnumerator(olapContext);
        default:
            throw new IllegalArgumentException("Wrong type " + type + "!");
        }
}

```

- OLAPEnumerator.queryStorage()

由OLAPTable.asQueryable进入，到了OLAPEnumerator.queryStorage()，终于能看到真实的查库操作了。

```
    private ITupleIterator queryStorage() {
        logger.debug("query storage...");

        // bind dynamic variables
        bindVariable(olapContext.filter);

        olapContext.resetSQLDigest();
        SQLDigest sqlDigest = olapContext.getSQLDigest();

        // query storage engine
        IStorageQuery storageEngine = StorageFactory.createQuery(olapContext.realization);
        ITupleIterator iterator = storageEngine.search(olapContext.storageContext, sqlDigest, olapContext.returnTupleInfo);
        if (logger.isDebugEnabled()) {
            logger.debug("return TupleIterator...");
        }

        return iterator;
    }

```

这里StorageEngine 由StorageFactory创建，且有三种不同的实现，默认还是HBase

```
    private static ThreadLocal<ImplementationSwitch<IStorage>> storages = new ThreadLocal<>();

    public static IStorage storage(IStorageAware aware) {
        ImplementationSwitch<IStorage> current = storages.get();
        if (storages.get() == null) {
            current = new ImplementationSwitch<>(KylinConfig.getInstanceFromEnv().getStorageEngines(), IStorage.class);
            storages.set(current);
        }
        return current.get(aware.getStorageType());
    }
    
   //KylinConfig.getInstanceFromEnv().getStorageEngines()
	public Map<Integer, String> getStorageEngines() {
        Map<Integer, String> r = Maps.newLinkedHashMap();
        // ref constants in IStorageAware
        r.put(0, "org.apache.kylin.storage.hbase.HBaseStorage");
        r.put(1, "org.apache.kylin.storage.hybrid.HybridStorage");
        r.put(2, "org.apache.kylin.storage.hbase.HBaseStorage");
        r.putAll(convertKeyToInteger(getPropertiesByPrefix("kylin.storage.provider.")));
        return r;
    }
   
```

- OLAPTableScan.register()

由于OLAPTable实现了TranslatableTable，它会通过一系列的relation operators将结果聚合,relation operators的注册逻辑在OLAPTableScan中。

```
 @Override
    public void register(RelOptPlanner planner) {
        // force clear the query context before traversal relational operators
        OLAPContext.clearThreadLocalContexts();

        // register OLAP rules
        planner.addRule(OLAPToEnumerableConverterRule.INSTANCE);
        planner.addRule(OLAPFilterRule.INSTANCE);
        planner.addRule(OLAPProjectRule.INSTANCE);
        planner.addRule(OLAPAggregateRule.INSTANCE);
        planner.addRule(OLAPJoinRule.INSTANCE);
        planner.addRule(OLAPLimitRule.INSTANCE);
        planner.addRule(OLAPSortRule.INSTANCE);
        planner.addRule(OLAPUnionRule.INSTANCE);
        planner.addRule(OLAPWindowRule.INSTANCE);
        ...
   }

```

这里着重看OLAPToEnumerableConverterRule 里返回的 OLAPToEnumerableConverter的实现，它是解释我在前言里提到的问题的关键。

- OLAPToEnumerableConverter.implement()

这里面有对所有满足query条件的realization选择的实现

```

   public Result implement(EnumerableRelImplementor enumImplementor, Prefer pref) {
    	  
    	 ...
        // identify model & realization
        List<OLAPContext> contexts = listContextsHavingScan();

        // intercept query
        List<QueryInterceptor> intercepts = QueryInterceptorUtil.getQueryInterceptors();
        for (QueryInterceptor intercept : intercepts) {
            intercept.intercept(contexts);
        }

		 //RealizationChooser 中有对Realization选择的具体实现
        RealizationChooser.selectRealization(contexts);
        ...

		 return impl.visitChild(this, 0, inputAsEnum, pref);
    }


```

- RealizationChooser.attemptSelectRealization()

attemptSelectRealization方法里面主要干了两件事： 1）拉取属于该project与factTableName下的所有Realization，经过一系列的条件过滤掉不符合query的Realization，并将符合条件的Realization按照RealizationCost排序。2）对第一步收集的Realization map，调用QueryRouter.selectRealization()，一旦QueryRouter.selectRealization()有返回值立即中断循环返回最终选择的Realization

- RealizationChooser.makeOrderedModelMap() 部分的实现逻辑如下：

```
       //按条件过滤realization
        for (IRealization real : realizations) {
            //过滤disabled cube
            if (real.isReady() == false) {
                context.realizationCheck.addIncapableCube(real,
                        RealizationCheck.IncapableReason.create(RealizationCheck.IncapableType.CUBE_NOT_READY));
                continue;
            }
            //过滤不包含querycontext里面全部的columns
            if (containsAll(real.getAllColumnDescs(), first.allColumns) == false) {
                context.realizationCheck.addIncapableCube(real, RealizationCheck.IncapableReason
                        .notContainAllColumn(notContain(real.getAllColumnDescs(), first.allColumns)));
                continue;
            }
            ／／过滤存在黑名单里面的cube
            if (RemoveBlackoutRealizationsRule.accept(real) == false) {
                context.realizationCheck.addIncapableCube(real, RealizationCheck.IncapableReason
                        .create(RealizationCheck.IncapableType.CUBE_BLACK_OUT_REALIZATION));
                continue;
            }

			  //过滤完，按RealizationCost排序
            RealizationCost cost = new RealizationCost(real);
            DataModelDesc m = real.getModel();
            Set<IRealization> set = models.get(m);
            if (set == null) {
                set = Sets.newHashSet();
                set.add(real);
                models.put(m, set);
                costs.put(m, cost);
            } else {
                set.add(real);
                RealizationCost curCost = costs.get(m);
                if (cost.compareTo(curCost) < 0)
                    costs.put(m, cost);
            }
        }

```

重点就在RealizationCost的实现里了

```
  public RealizationCost(IRealization real) {
            // ref Candidate.PRIORITIES
            this.priority = Candidate.PRIORITIES.get(real.getType());

            // ref CubeInstance.getCost()
            int c = real.getAllDimensions().size() * CubeInstance.COST_WEIGHT_DIMENSION
                    + real.getMeasures().size() * CubeInstance.COST_WEIGHT_MEASURE;
            for (JoinTableDesc join : real.getModel().getJoinTables()) {
                if (join.getJoin().isInnerJoin())
                    c += CubeInstance.COST_WEIGHT_INNER_JOIN;
            }
            this.cost = c;
        }


```

到此，对于kylin的Realization的成本计算规则清楚了。就是对dimension，measure,jointable三个维度的数量进行加权求和，得到的就是每个Realization对应的成本。相对的，每个维度对应的权重是有所斟酌的，dimension对应的是10，measure为1（考虑到是预计算的结果），jointable为100。从这也能看出建模时应该考虑的优化方向：避免过多的dimension，以及jointable的操作，结果尽量从预计算中出。

## 总结

这次经过对kylin query源码的分析，基本上对kylin的核心代码都过了一遍，学习了不少优秀的代码解耦方式，也对底层原理加深了理解。关于RealizationCost的实现，目前kylin实现比较简单，在遍历所有满足条件的实现时找到Realization便返回处理的有些过于仓促。对于其是map的结构或许kylin在之后的扩展方面也是有所考虑。目前我们还不打算扩展Realizaiton的选择策略，了解了源码就可以在建模层面将查询结果不如意的情况给规避了。
