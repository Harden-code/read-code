### 缓存

MyBatis提供了一个配置参数localCacheScope，用于控制一级缓存的级别，该参数的取值为SESSION、STATEMENT，当指定localCacheScope参数值为SESSION时，缓存对整个SqlSession有效，只有执行DML语句（更新语句）时，缓存才会被清除。当localCacheScope值为STATEMENT时，缓存仅对当前执行的语句有效，当语句执行完毕后，缓存就会被清空。

一级缓存只能控制级别不能关闭；二级缓存通过在mapper里< cache>配置；

Cache 接口默认实现org.apache.ibatis.cache.impl.PerpetualCache ，其他实现都是在原有基础上进行包装实现

#### 一级缓存实现

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    }
	...
      //现在缓存里进行查找
  list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
	...
    //STATEMENT 查询完直接清除
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      // issue #482
      clearLocalCache();
    }
  return list;
}
```

####二级缓存实现

SqlSession将执行Mapper的逻辑委托给Executor组件完成，而Executor接口有几种不同的实现，分别为SimpleExecutor、BatchExecutor、ReuseExecutor。另外，还有一个比较特殊的CachingExecutor，CachingExecutor用到了装饰器模式，在其他几种Executor的基础上增加了二级缓存功能。

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
  } else {
    executor = new SimpleExecutor(this, transaction);
  }
  //装饰器
  if (cacheEnabled) {
    executor = new CachingExecutor(executor);
  }
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}
```

###Mybatis懒加载和级联映射

####级联操作

级联映射实现一对多，一对一或者多对多的关联查询

一对多配置：通过<resultMap>配置<colleciton property=""  select=>标签

一对一配置：通过<resultMap>配置< association property="">< id column=" " properties="" >< /association >标签

####懒加载

懒加载，当在一个实体对象中关联其他实体对象时，如果不需要获取被关联的实体对象，则不需要为被关联执行额外查询操作，仅当调用当前实体对象get方法获取被关联实体对象时，才会执行一次额外的查询操作

开启方式：

- MyBatis主配置 文件中< settings>提供了lazyLoadingEnabled和aggressiveLazyLoading参数用来控制是否开启懒加载机制。
- < collection>和< association>标签还提供了一个fetchType属性，用于控制级联查询的加载行为，fetchType属性值为lazy时表示该级联查询采用懒加载方式，当fetchType属性值为eager时表示该级联查询采用积极加载方式。



### Mapper动态代理对象注册

@MapperScan