---
title: MyBatis源码分析（5）- 缓存的实现原理(1)-一级缓存
date: 2017/12/8
updated: 2017/12/20
tags:
   - javaee
   - mybatis
categories:
   - Programming Language
   - Java
   - Framework
   - MyBatis
---
在Mybatis的[文档](http://www.mybatis.org/mybatis-3/sqlmap-xml.html#cache)中，明确写出Mybatis有二级缓存。
>MyBatis includes a powerful transactional query caching feature which is very configurable and customizable. A lot of changes have been made in the MyBatis 3 cache implementation to make it both more powerful and far easier to configure.
By default, just local sessión caching is enabled that is used solely to cache data for the duration of a sessión. To enable a global second level of caching you simply need to add one line to your SQL Mapping

---
>Mybatis包含一个强大的事务查询缓存，Mybatis3做了很多改变使得它更加强大和易于配置。 默认情况下，只有`local cache`是开启的，这个缓存仅仅在一个`Sqlsession`的生命周期里有效。 你必须手动开启二级缓存。

那么这两级缓存分别是如何实现的呢？ 本节分析下默认的缓存 - `local session`的具体实现。 事实上，这个缓存默认是打开的，并且没有配置可以关闭。
<!--more-->

前面已经说过，Mybatis通过`SqlSession`接口的调用其实都是委托`Executor`接口处理的。具体可以参考前面几节。 缓存也是一样，也都在`Executor`接口中。 该接口的继承关系如图:
![]( https://gitee.com/angus_lean/markDownPic/raw/master/2017/11/mybatis-source-analyze-executor.png)
所以我们这里直接看`BaseExecutor`抽象类，该类提供了1级缓存的处理。


## 1. `BaseExecutor`抽象类
在[前面章节](https://anguslean.github.io/2017/12/07/Mybatis%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%883%EF%BC%89-%20SqlSession%E6%8E%A5%E5%8F%A3%E5%A2%9E%E5%88%A0%E6%94%B9%E6%9F%A5%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86(1)/)中说过，`SqlSession`接口的实现实际上转给`Executor`接口的。缓存也是一样。 而在`Executor`接口继承体系中，`BaseExecuto`抽象类提供了大部分的基础功能，仅仅是`doUpdate,doQuery`方法交由子类实现。 如何的代码清晰的展示了缓存的使用：   

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    //创建一个缓存对象
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    //传递缓存对象给子方法
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

上面的代码创建了一个`CacheKey`实例，然后传递给`query`方法。 我们再看`query`方法：

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        //根据传来的缓存对象查看当前实例的缓存Map中是否有对应的值，如果没有才走到查询数据库去
        queryStack++;
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        //清除缓存
        // issue #601
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}
```

其中，`localCache`是一个`PerpetualCache`类的实例。其内部实现就是一个`HashMap`， 并非是线程安全的。 同时，`localCache`是实例属性，而不是类静态属性，所以说这个缓存就是`SqlSession`实例级别，生命周期同`SqlSession`实例一样。 
现在我们清除了Mybatis的一级缓存实际上就是通过一个`HashMap`保存查询结果，在下次查询时，先判断map中是否有当前`CacheKey`的键值，有的话直接返回。 这同我们平时使用`Map`作为缓存并无区别。

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        localCache.removeObject(key);
    }
    //更新缓存
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

那么，对于一个`SqlSession`的不同SQL执行、相同SQL不同参数调用以及如果有数据更改的情况，Mybatis是如何处理的？

### 1.1 查询缓存处理
上面的`query`方法中有一步`createCacheKey`，这里创建了一个缓存键值， 它最终被拿来到缓存Map中取值，所以说这个*Key*的创建是关键，它决定了当前这次查询是否复用以前的查询结果。 下面看代码:

```java
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    //根据查询参数的ID，查询偏移量，SQL创建了一个CacheKey实例
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // 处理SQL中的参数映射
    for (ParameterMapping parameterMapping : parameterMappings) {
        if (parameterMapping.getMode() != ParameterMode.OUT) {
            Object value;
            String propertyName = parameterMapping.getProperty();
            if (boundSql.hasAdditionalParameter(propertyName)) {
                value = boundSql.getAdditionalParameter(propertyName);
            } else if (parameterObject == null) {
                value = null;
            } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                value = parameterObject;
            } else {
                MetaObject metaObject = configuration.newMetaObject(parameterObject);
                value = metaObject.getValue(propertyName);
            }
            cacheKey.update(value);
        }
    }
    if (configuration.getEnvironment() != null) {
        cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
}
```

上面多次调用到了`CacheKey`的`update`方法， 我们看下`CacheKey`的实现：
```java
public void update(Object object) {
    //简单的说，update方法保存了当前这个缓存实例所有的update参数
    //以及根据参数的添加不断的更新hashCode。这样使得后面如果有完全一样的请求过来
    //可以保证这两个CacheKey对象的hashCode一样
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);
    count++;
    checksum += baseHashCode;
    baseHashCode *= count;
    hashcode = multiplier * hashcode + baseHashCode;
    updateList.add(object);
}
```
再看下很重要的`equals`方法的实现:

```java
public boolean equals(Object object) {
    if (this == object) {
        return true;
    }
    if (!(object instanceof CacheKey)) {
        return false;
    }
    //判断上面不断叠加的hashCode以及checkSum以及调用update的长度。
    final CacheKey cacheKey = (CacheKey) object;
    if (hashcode != cacheKey.hashcode) {
        return false;
    }
    if (checksum != cacheKey.checksum) {
        return false;
    }
    if (count != cacheKey.count) {
        return false;
    }
    //判断所有的Update的参数是否相等
    for (int i = 0; i < updateList.size(); i++) {
        Object thisObject = updateList.get(i);
        Object thatObject = cacheKey.updateList.get(i);
        if (!ArrayUtil.equals(thisObject, thatObject)) {
            return false;
        }
    }
    return true;
}
```
毫无疑问，`hashCode`方法是返回上面的`hashcode`变量:
```java
public int hashCode() {
    return hashcode;
}
```


#所以说，Mybatis提供的一级缓存就是通过一个`Map`来存储一个`SqlSession`的所有查询结果，通过参数名、数量等等来确定一个缓存的Key。#

那么问题来了，对于同一个`SqlSession`来说，相同的查询会被缓存。但是一个实例它不知道这段时间数据库是不是有改变，所以**如果在多线程环境下使用多个`SqlSession`，多个`SqlSession`操作同一张表，那么是完全可能出现数据延迟的,也就是说查询不到最新数据**,甚至说，**在一个`SqlSession`的生命期内，创建另一个`SqlSession`实例，是不能保证数据的实时性的**。 这也说明了为什么[官方文档](http://www.mybatis.org/mybatis-3/getting-started.html)说`SqlSession`的实例不能被长期持有，它应该是方法级别的。 总结来说，对于纯粹使用**Mybatis**库的人来说(指不是使用其他DI包装库，如mybatis-spring)，如下几点必须清楚:
- **`SqlSession`不是线程安全的。 它不应该、更不能在多个线程间共享。 它的引用更不能被长期持有，必须使用完就关闭。**
- **默认情况下，Mybatis会打开`SqlSession`级别缓存，如果说严格遵守上面规则的话问题不大，但是必须要知道一个`SqlSession`实例会缓存查询结果，在缓存期间，有任何其他方式（其他的`SqlSession`实例或者直接更改数据库）变更了数据库数据时，这个实例不会刷新缓存。 所以这里必须根据自己的业务灵活处理。**

### 1.2 更新缓存处理
上面说清楚了在查询时的缓存处理情况，下面看下在涉及到更新数据的处理情况
```java
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    //直接清除缓存
    clearLocalCache();
    return doUpdate(ms, parameter);
}
```
明显看到，在涉及到更新操作时，一个`SqlSession`实例会直接删除所有缓存。

### 1.3 如何关闭一级缓存
根据上面的代码可以看出，这一级的缓存是没有通过配置项来控制开关的。 所以说，这一级缓存不能被关闭， 也不应该被关闭。 而这一级缓存导致的上面说的多个`SqlSession`实例同时存在的情况下，数据不会刷新的情况，也应该由使用者遵循**使用时获取，不用就关闭**的原则来避免。


### 1.4 总结
Mybatis提供了二级缓存，其中一级缓存是`SqlSession`级别的，它会缓存当前实例的所有查询结果，对于后续的相同查找会直接返回缓存结果。 同时，`SqlSession`不是线程安全的，并且它的缓存在有外部方式更改数据时不会刷新，使用时务必注意。