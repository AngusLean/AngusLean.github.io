---
title: MyBatis源码分析（6）- 缓存的实现原理(2)-二级缓存
date: 2017/12/12
updated: 2019/4/19
tags:
   - javaee
   - mybatis
categories:
   - Programming Language
   - Java
   - Framework
   - MyBatis
---
在上一节分析了Mybatis的一级缓存的实现原理，也就是一个HashMap保存结果。 本节尝试分析下二级缓存的原理。
上章提到的`Executor`接口的继承关系中还有一个很重要的类`CacheExecutor`没有介绍，该类其实就是一个委派类。 他提供了Mybatis中的二级缓存功能，然后把实际查询委派给实际处理类。

<!--more-->

![](https://gitee.com/angus_lean/markDownPic/raw/master/2017/11/mybatis-source-analyze-executor.png)


## 1. `CacheExecutor`类
我们首先看看最重要的`query, update`两个方法。

### 1.1 `query`方法
```java

public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds,   ResultHandler resultHandler) throws SQLException {
   BoundSql boundSql =ms.getBoundSql(parameterObject);
   //创建一个缓存
   CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
   //使用该缓存去查询
   return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}

//实际查询方法
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
    //判断当前的mappedstatement是否需要刷新缓存
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, parameterObject, boundSql);
            //获取缓存数据
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
            //调用被代理类的query方法获取数据并且缓存
                list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            return list;
        }
    }
    //调用被代理类的query方法获取数据
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

---

### 1.2 `update`方法
再看看`update`方法的实现
```java
public int update(MappedStatement ms, Object parameterObject) throws SQLException {
//判断当前的mappedstatement是否需要刷新缓存
    flushCacheIfRequired(ms);
    //调用被代理类的update方法获取数据
    return delegate.update(ms, parameterObject);
}
```
update也是直接交给被委派的类处理。
到这里我们可以看到，查询和更新方法的大概步骤都与上节介绍的`BaseExecutor`中的实现类似。 都是如下流程：
- 创建一个缓存Key
- 根据这个key到某个容器中获取对应的数据
- 获取不到的话交给其他部分查询（这里是被委派的类，BaseExecutor中是handler接口）
- 再缓存数据并返回

这两个类的缓存的key都是`BaseExecutor`接口中的`createCacheKey`方法生成的`CacheKey`实例，它们的实际区别有如下几个：

- 保存缓存的容器
- `BaseExecutor`中的缓存不可关闭，而`CacheExecutor`是可以关闭的
- `BaseExecutor`中缓存直接通过`CacheKey`就可以查询，而`CacheExecutor`同时需要`CacheKey`以及`Cache`

---

## 2 二级缓存的真正实现
### 2.1 基本实现
上面说道，`CacheExecutor`接口保存缓存的容器是不同于`BaseExecutor`的。 那么它是使用了一个什么样的容器呢？
它的缓存容器的类型是`TransactionalCacheManager`。 我们看看具体实现：
```java
public class TransactionalCacheManager {
    private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<Cache, TransactionalCache>();

    public void clear(Cache cache) {
        getTransactionalCache(cache).clear();
    }
//获取缓存不同于BaseExecutor接口
    public Object getObject(Cache cache, CacheKey key) {
        return getTransactionalCache(cache).getObject(key);
    }

    public void putObject(Cache cache, CacheKey key, Object value) {
        getTransactionalCache(cache).putObject(key, value);
    }
     private TransactionalCache getTransactionalCache(Cache cache) {
        TransactionalCache txCache = transactionalCaches.get(cache);
        if (txCache == null) {
          txCache = new TransactionalCache(cache);
          transactionalCaches.put(cache, txCache);
        }
        return txCache;
     }
    //...忽略其他代码
}
```
该缓存容器其实是包装了另外一个名为`TransactionalCache`的类，通过`Cache`类型的实例为键值来保存数据。 可以看到，在向该缓存容器查询数据时，需要提供2个参数 `Cache cache, CacheKey key`, 通过前者`cache`得到一个`TransactionalCache`然后再通过`key`取得实际的缓存结果。 我们看看`TransactionalCache`的实现
```java
public class TransactionalCache implements Cache {

    private final Cache delegate;
    private boolean clearOnCommit;
    private final Map<Object, Object> entriesToAddOnCommit;
    private final Set<Object> entriesMissedInCache;
//内部委派一个Cache实例。 事实上这个实例才是真正的缓存内容保存的地方， 当前类只是在提交时才刷新缓存到delegate上
    public TransactionalCache(Cache delegate) {
        this.delegate = delegate;
        this.clearOnCommit = false;
        this.entriesToAddOnCommit = new HashMap<Object, Object>();
        this.entriesMissedInCache = new HashSet<Object>();
    }

    @Override
        public String getId() {
        return delegate.getId();
    }

    @Override
        public int getSize() {
        return delegate.getSize();
    }

    @Override
        public Object getObject(Object key) {
        Object object = delegate.getObject(key);
        if (object == null) {
            entriesMissedInCache.add(key);
        }
    }
//添加一个缓存只是把它放到了一个Map中，没有提交给真正的cache
    @Override
        public void putObject(Object key, Object object) {
        entriesToAddOnCommit.put(key, object);
    }

//提交时才刷新当前所有缓存到delegate上
    public void commit() {
        if (clearOnCommit) {
            delegate.clear();
        }
        flushPendingEntries();
        reset();
    }

    private void flushPendingEntries() {
        for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
            delegate.putObject(entry.getKey(), entry.getValue());
        }
        for (Object entry : entriesMissedInCache) {
            if (!entriesToAddOnCommit.containsKey(entry)) {
                delegate.putObject(entry, null);
            }
        }
    }
//移除了部分非关键代码
}
```
通过上面的代码分析，我们可以看出MyBatis的二级缓存的实现和一级缓存其实区别不是很大：
1. 通过一个Map<Cache, TransactionalCache>持有所有的缓存。Cache每一个MappedStatement都不相同。
2. 添加缓存时，通过一个TransactionalCache实例委派真正的Cache实现， 所有添加到缓存的内容都先保存在TransactionalCache实例中，直到commit方法被调用，所有保存的全部刷新到委派类中并且删除本地的。
3. 查询时，首先根据每一个`MappedStatement`的实例取得对应的`Cache`的实例，然后在`Map<Cache, TransactionalCache>`获取TransactionalCache实例，该实例再从委派的`Cache`的实例从通过`CacheKey`获取缓存结果

---
### 2.2 `Cache`接口的几种实现
上面说道，`Cache`实例保存了相同条件的`MappedStatement`的所有查询结果。 `Cache`接口有如下的实现：

- SynchronizedCache
- WeakCache
- TransactionalCache
- SoftCache
- SerializedCache
- ScheduledCache
- LruCache
- FifoCache
- BlockingCache

每个实现根据名称就可以看出原理，就不赘述。 MyBatis默认使用的`LruCache`这个实现，默认情况下长度为1024.可以通过在mapper xml中的<cache/>设置。
而每个`MappedStatement`中的`Cache`是通过`MappedStatement.Builder`方法设置，而这个则会读取用户配置的缓存然后实例化一个`Cache`,可以参考`MappedStatement`类。这就完成了整个二级缓存的步骤。

---

### 2.3 二级缓存的开关
首先，这个二级缓存默认是关闭的，可以在mapper xml中通过添加

```
<cache
  eviction="FIFO"
  flushInterval="60000"
  size="512"
  readOnly="true"/>
```

这种标记来打开二级缓存，具体参考文档。关闭这个缓存在mybatis config文件中，`setting`节中添加如下字段（这是全局开关）

```
<setting name="cacheEnabled" value="true"/>
```
