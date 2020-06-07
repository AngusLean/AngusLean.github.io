---
title: MyBatis源码分析（3）- SqlSession接口增删改查的实现原理(1)
date: 2017/11/30
updated: 2018/2/9
tags:
   - javaee
   - mybatis
categories:
   - Programming Language
   - Java
   - Framework
   - MyBatis
---
上篇文章说明了mapper接口最终是调用了`SqlSession`的xml调用，那么Mybatis是如何执行mapper xml的， 一个xml是如何解析以及最终传达到数据库的，本节分析这个问题。

<!--more-->

## 1. 开始
第一篇文章中列出的Mybatis中最终的几个类来说，上篇我们已经大概讲了`Configuration`类的一个功能- 连接mapper interface与mapper xml 。 而关于如何读取配置文件和解析mapper xml相对而言不是那么重要，mybatis在知道需要调用的mapper xml的的方法之后，是如何最终影响到数据库的才是最重要的。简单的说，就是
```java
SqlSession session = sqlSessionFactory.openSession();
try {
//查找
 List<Post> posts = session.selectList("org.apache.ibatis.domain.blog.mappers.PostMapper.findPost");
//新增
session.insert("org.apache.ibatis.domain.blog.mappers.AuthorMapper.insertAuthor", expected);
} finally {
  session.close();
}

```
是怎么到达数据库的。
本节尝试理清这个逻辑。

## 2. 实现原理
### 2.1 `SqlSession`接口
首先我们看下`SqlSession`这个接口的实现类


![](https://gitee.com/angus_lean/markDownPic/raw/master/2017/11/mybatis-source-analyze-sqlsession.png)

可以看到该类共有2个实现，其中`DefaultSqlSession`是实际处理逻辑的一个类，而`SqlSessionManager`如同其名，它只是在某方面自动配置而已，实际的逻辑处理最终还是会交回`DefaultSqlSession`。
#### 2.1.1 `SqlSessionManager`
`SqlSessionManager`所有的操作都有自己的成员变量`sqlSessionProxy`处理，而`sqlSessionProxy`是通过`SqlSessionInterceptor`生成的一个代理。它的`invoke`方法是：
```java
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
      if (sqlSession != null) {
        try {
          return method.invoke(sqlSession, args);
        } catch (Throwable t) {
          throw ExceptionUtil.unwrapThrowable(t);
        }
      } else {
        final SqlSession autoSqlSession = openSession();
        try {
          final Object result = method.invoke(autoSqlSession, args);
          autoSqlSession.commit();
          return result;
        } catch (Throwable t) {
          autoSqlSession.rollback();
          throw ExceptionUtil.unwrapThrowable(t);
        } finally {
          autoSqlSession.close();
        }
      }
    }
```
可以看到，这个manager可以说就是默认实现了提交和事物回滚，不需要再手动调用。 但是实际处理依然是通过`Configuration`类获取的`SqlSession`实例， 这个实例事实上就是`DefaultSqlSession`类型。
#### 2.1.2 `DefaultSqlSession`
对于这个最重要的类，我们直接选取一个方法的实现来看：
```java
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
对于所有select,最终都会是调用上面的方法。 然后对于返回结果是`Map`等等再分别处理。而对于`update`来说：
```
public int update(String statement, Object parameter) {
    try {
      dirty = true;
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
而`delete`方法则是：
```
public int delete(String statement) {
    return update(statement, null);
  }
```
相当于还是调用的update方法。
可以看出一个很明显的共性就是如下3步：
- 通过`Configuration`类获取`MappedStatement`， 即mapper xml的一个抽象实体
- 包装请求参数
- 调用`Executor`的相应方法
关于`Configuration`解析以及获取`MappedStatement`.留到后面再说。 这里重点是`Executor`接口

### 2.2 `Executor`接口
#### 2.2.1 整体结构

![](https://gitee.com/angus_lean/markDownPic/raw/master/2017/11/mybatis-source-analyze-executor.png)


这些类的大概介绍：
- `Executor`接口   提供了对数据库的`update`,`query`,`commit`,`rollback`等方法。
- `CacheExecutor`类   一个`Executor`实例的委派类，添加了查询结果的缓存。
- `BaseExecutor`抽象类
这个类比较重要， 在`Executor`继承体系里他有3个子类，所以毫无疑问它提供了一些公告的方法。
在 2.1.2 `DefaultSqlSession` 节中粘贴的代码显示，`DefaultSqlSession`中的`update, insert, delete, select`最终都会调用某个`Executor`接口实现类中的`select, update`方法。而这两个方法事实上就是在`BaseExecutor`中实现的：

```java
//所有的insert,update都会调用到这里来
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    //doUpdate是抽象方法。
    return doUpdate(ms, parameter);
}

//DefaultSqlSession调用的就是这个方法
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    //在创建缓存过后再调用下面的方法
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}

//真正的query的实现
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
        queryStack++;
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            //看名字就知道是和数据库紧密相关，应该就是执行SQL的关键位置
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }

        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            clearLocalCache();
        }
    }
    return list;
}
```

---
```java
  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
//doQuery是一个抽象方法
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```
上面的4个方法就是`BaseExecutor`接口提供的实现，但是其中具体的需要和数据库打交道的`doQuery,doUpdate`方法留给了子类去实现。当然，这里先不管上面处理缓存的部分代码。
- `BatchExecutor`  批量执行， 对应着JDBC的批量执行的操作。 这个后面有空再理理。
- `SimpleExecutor` 关键类  下篇分析

## 3 其他
问题：
- 用到的设计模式
- 用到的OO思想

